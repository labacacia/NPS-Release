English | [中文版](./NPS-2-NWP.cn.md)

# NPS-2: Neural Web Protocol (NWP)

**Spec Number**: NPS-2  
**Status**: Proposed  
**Version**: 0.10  
**Date**: 2026-05-01  
**Port**: 17433 (default, shared) / 17434 (optional dedicated)  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.6), NPS-3 (NIP v0.6), NPS-4 (NDP v0.6)  

> This document is the NWP detailed specification. For a suite overview see [NPS-0-Overview.md](NPS-0-Overview.md).

---

## 1. Terminology

Keywords "MUST", "MUST NOT", "REQUIRED", "SHOULD", "MAY" in this document are interpreted per RFC 2119.

---

## 2. Protocol Overview

NWP defines how AI Agents access web data and services. Agents use `nwp://` addresses to reach three types of Neural Nodes (Memory / Action / Complex). Node responses are directly machine-understandable, requiring no semantic parsing layer.

### 2.1 Node Types

| Type | Responsibility | Typical Data Sources |
|------|---------------|---------------------|
| **Memory Node** | Data storage and retrieval, no compute logic | RDS, NoSQL, file systems, vector databases |
| **Action Node** | Executes operations, returns results or side effects | Functions, external APIs, message queues, Webhooks |
| **Complex Node** | Mixed data and operations, with sub-node references | All of the above + sub-node references |
| **Anchor Node** | Cluster control plane and external entry point — routes inbound NWP `Action`/`Query` frames to member nodes via NOP, optionally maintains member-node topology | AaaS platforms, multi-agent service gateways, sub-cluster routers |
| **Bridge Node** | Translates between NPS frames and non-NPS protocols (HTTP/HTTPS, gRPC, MCP, A2A) | Calls to legacy REST APIs, gRPC services, Model Context Protocol servers, Agent-to-Agent endpoints |

A node MAY simultaneously carry more than one role (for example, a single deployment can be both a Memory Node and an Anchor Node when role separation is unnecessary). Multi-role declaration travels in the NDP `Announce` frame's `node_roles` field (NPS-4 §3.1).

> **Anchor Node** and **Bridge Node** were introduced together by [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md), replacing the original `Gateway Node` type:
> - **Anchor Node** inherits the cluster-entry / NOP-routing role that Gateway Node was carrying. It is stateless per request but MAY maintain a long-lived registry of member nodes.
> - **Bridge Node** is a new type whose role is **NPS → external protocol** translation. It is stateless per request and does not participate in cluster topology. (Direction note: this is the inverse of the bridges historically published under `compat/*-bridge`, which are external-protocol → NPS ingress adapters; those have been renamed `compat/*-ingress` to free the "Bridge" word for this new node type.)
> - The original `Gateway Node` term is retired; the wire value `"gateway"` is removed and parsers MUST reject it with a clear error referencing CR-0001.

#### Removed types

> **Gateway Node** (removed in v1.0-alpha.3) — Split into **Anchor Node** (cluster entry / NOP routing) and **Bridge Node** (NPS↔non-NPS protocol translation). See [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md) for full rationale and migration notes. Implementations MUST reject the legacy `node_type: "gateway"` with `NWP-MANIFEST-NODE-TYPE-REMOVED` and the legacy `node_roles: ["gateway"]` (or legacy `node_kind: "gateway"`) with `NDP-ANNOUNCE-ROLE-REMOVED`; response SHOULD include a `hint` pointing to NPS-CR-0001.

#### Node Role Resolution

Two fields participate in node-role declaration at different protocol layers. They are intentionally distinct in name and semantics:

| Field | Protocol | Layer | Cardinality | Authoritative for |
|-------|----------|-------|-------------|-------------------|
| `node_roles` | NDP `Announce` (NPS-4 §3.1) | Discovery | Array — all roles the node carries | Full role set; used for discovery filtering and cluster membership |
| `node_type` | NWP NWM (§4.1) | Service | String — single operative role | Which role this particular `/.nwm` endpoint is serving |

**Constraint**: `node_type` MUST be one of the values declared in the node's `node_roles`. Validators SHOULD verify this against cached NDP `Announce` data when available.

Multi-role nodes (e.g., `node_roles: ["anchor", "memory"]`) MAY expose separate `/.nwm` endpoints on different paths or ports, each advertising a different `node_type`; or may choose a single dominant `node_type` matching the primary role served at that endpoint. In either case the constraint holds: `node_type ∈ node_roles`.

#### Anchor Node — detailed semantics

An Anchor Node MUST:

1. Accept inbound NWP `Action` and `Query` frames addressed to the cluster (i.e. to the Anchor's NID rather than to a specific member NID).
2. Dispatch frames to appropriate member nodes based on their declared capabilities and current load. The reference dispatch path converts an `ActionFrame` into a NOP `TaskFrame` and delegates to a local NOP orchestrator (see [NPS-AaaS-Profile §2](services/NPS-AaaS-Profile.md)).
3. Aggregate outbound responses from member nodes into a single response stream toward the originating caller.
4. Optionally maintain a registry of member nodes within its cluster (their NIDs, declared capabilities, and `activation_mode` per [NPS-Node Profile §6](services/NPS-Node-Profile.md)). Member nodes register on cluster join via NDP `Announce` frames carrying `cluster_anchor` referencing the Anchor Node's NID; deregistration follows standard NDP offline semantics.

A cluster MUST have at least one Anchor Node. High-availability deployments MAY operate multiple Anchor Nodes for the same cluster; the consensus protocol between Anchor Nodes is implementation-defined and is deferred to NPS-AaaS Profile L3.

Anchor Nodes that maintain a member registry MUST expose it via the reserved query types `topology.snapshot` and `topology.stream` (§12), per [NPS-CR-0002](cr/NPS-CR-0002-anchor-topology-queries.md). Both are mandatory at NPS-AaaS Profile L2 and above.

#### Bridge Node — detailed semantics

A Bridge Node MUST:

1. Accept inbound NWP frames carrying a `bridge_target` parameter that identifies the external protocol and endpoint (concrete schema for `bridge_target` is implementation-defined in this CR; standardization is deferred to a follow-up CR per protocol).
2. Produce outbound requests in the target protocol's format.
3. Translate target-protocol responses back into NWP frames (typically `CapsFrame`).

Bridge Nodes are stateless per request and do not participate in cluster topology. A single Bridge Node MAY translate to multiple distinct external protocols; deployments MAY operate dedicated Bridge Nodes per protocol for isolation.

Standard external protocols expected to be supported by reference Bridge Node implementations:

- HTTP / HTTPS (REST and streaming)
- gRPC (unary and streaming)
- MCP (Model Context Protocol)
- A2A (Agent-to-Agent protocol)

Additional protocol adapters MAY be registered through future CRs. The set of supported protocols is declared in NDP `Announce.bridge_protocols` (NPS-4 §3.1).

### 2.2 Overlay Mode

Attach a NWP interface to an existing HTTP service. The server distinguishes visitors by request headers:

```
Request contains X-NWP-Agent or HelloFrame  →  return application/nwp-*
Regular browser request (no above markers)  →  return text/html (normal website)
```

In Overlay mode, NWP uses HTTP transport with frames serialized in the HTTP body. See [NPS-1-NCP.md §2.2](NPS-1-NCP.md#22-transport-modes).

---

## 3. Node Address Specification

### 3.1 nwp:// URL Syntax (ABNF)

```abnf
nwp-url     = "nwp://" host [":" port] "/" node-path ["/" sub-path]
host        = <RFC 3986 host>
port        = 1*DIGIT               ; default 17433
node-path   = segment *("/" segment)
sub-path    = "query" / "stream" / "invoke" / "subscribe" / "actions"
            / ".schema" / ".nwm"
segment     = 1*(ALPHA / DIGIT / "-" / "_")
```

### 3.2 Sub-Path Conventions

| Sub-Path | Method | Node Types | Description |
|----------|--------|-----------|-------------|
| `/query` | POST | Memory | Single structured query (returns CapsFrame) |
| `/stream` | POST | Memory | Streaming query (returns StreamFrame sequence) |
| `/invoke` | POST | Action / Complex | Operation invocation endpoint |
| `/subscribe` | POST | Memory | Change subscription endpoint (HTTP mode, SSE) |
| `/actions` | GET | Action / Complex | List callable operations (returns NWM actions subset JSON) |
| `/.schema` | GET | All | Schema definition (returns AnchorFrame JSON) |
| `/.nwm` | GET | All | Full node manifest (returns NWM JSON) |

---

## 4. Neural Web Manifest (NWM)

Every node MUST expose a machine-readable manifest at `/.nwm`, MIME type: `application/nwp-manifest+json`.

### 4.1 Complete Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `nwp` | string | Required | NWP version, currently `"0.4"` |
| `node_id` | string | Required | Node NID, format: `urn:nps:node:{host}:{path}` |
| `node_type` | string | Required | `"memory"` / `"action"` / `"complex"` / `"anchor"` / `"bridge"`. The legacy value `"gateway"` was removed in v1.0-alpha.3 (see §2.1 *Removed types* and [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md)); parsers MUST reject it with `NWP-MANIFEST-NODE-TYPE-REMOVED`. Any other unrecognized value MUST be rejected with `NWP-MANIFEST-NODE-TYPE-UNKNOWN`. This field declares the **operative role** this NWP service endpoint is serving — it MUST be one of the values in the node's NDP `Announce.node_roles` (NPS-4 §3.1). For multi-role nodes, `node_type` selects which role is active at this endpoint; see §2.1 *Node Role Resolution* for the full constraint. |
| `display_name` | string | Optional | Human-readable node name |
| `manifest_version` | string | Optional | Manifest version identifier (ETag), for conditional request cache control |
| `min_agent_version` | string | Optional | Minimum NPS version the Agent must support, format `"major.minor"`; Agents below this version MUST be rejected with `NWP-MANIFEST-VERSION-UNSUPPORTED` |
| `min_assurance_level` | string | Optional | One of `"anonymous"` (default) / `"attested"` / `"verified"` (see [NIP §5.1.1](NPS-3-NIP.md#511-assurance-levels-nps-rfc-0003)). Requests presenting a level lower than this MUST be rejected with `NWP-AUTH-ASSURANCE-TOO-LOW` (`NPS-AUTH-FORBIDDEN`); response SHOULD include a `hint` pointing to a CA enrolment URL. Default `"anonymous"` preserves backward compatibility with v1.0-alpha.2 nodes. Per-action override is permitted via the `min_assurance_level` field on an individual `ActionSpec` (§4.6). (NPS-RFC-0003) |
| `wire_formats` | array | Required | Supported encoding formats: `["ncp-capsule", "msgpack", "json"]` |
| `preferred_format` | string | Required | Preferred format |
| `schema_anchors` | object | Optional | Pre-declared schema anchors, `{name: anchor_id}` |
| `capabilities` | object | Required | Node capability declarations, see §4.2 |
| `data_sources` | array | Optional | Underlying data source identifier list |
| `auth` | object | Required | Authentication requirements, see §4.3 |
| `rate_limits` | object | Optional | Rate limit declarations, see §4.4 |
| `actions` | object | Conditionally Required | MUST be populated for Action/Complex nodes; operation registry, see §4.6 |
| `endpoints` | object | Required | URLs for each functional endpoint |
| `graph` | object | Optional | Sub-node references (Complex Node only), see §11 |
| `tokenizer_support` | array | Optional | List of tokenizers supported by the node (see [token-budget.md](token-budget.md)) |

### 4.2 capabilities Field

| Capability Key | Type | Description |
|---------------|------|-------------|
| `query` | bool | Supports QueryFrame (single query) |
| `stream_query` | bool | Supports streaming queries (StreamFrame response) |
| `aggregate` | bool | Supports aggregation queries (QueryFrame.aggregate) |
| `subscribe` | bool | Supports change subscriptions (DiffFrame push) |
| `subscribe_filter` | bool | Supports subscriptions with filter conditions |
| `vector_search` | bool | Supports vector similarity search |
| `token_budget_hint` | bool | Supports trimming responses based on CGN budget |
| `ext_frame` | bool | Supports extended frame header (large frame mode) |
| `e2e_enc` | bool | Supports NCP E2E encryption (ENC=1, see NPS-1-NCP §7.4) |
| `inline_anchor` | bool | Supports returning updated AnchorFrame inline in responses |

### 4.3 auth Field

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `required` | bool | Required | Whether authentication is required |
| `identity_type` | string | Conditionally Required | `"nip-cert"` / `"bearer"` / `"none"` |
| `trusted_issuers` | array | Conditionally Required | List of trusted CA URLs (required when identity_type is nip-cert) |
| `required_capabilities` | array | Optional | Capabilities the Agent MUST hold, e.g. `["nwp:query"]` |
| `scope_check` | string | Optional | Scope validation mode: `"prefix"` (default) / `"exact"` |
| `ocsp_url` | string | Optional | OCSP validation endpoint |

### 4.4 rate_limits Field

| Field | Type | Description |
|-------|------|-------------|
| `requests_per_minute` | uint32 | Max requests per Agent per minute |
| `requests_per_day` | uint32 | Max requests per Agent per day |
| `max_concurrent_streams` | uint32 | Max concurrent streams per Agent |
| `max_subscriptions` | uint32 | Max concurrent subscriptions per Agent |

### 4.5 Complete NWM Example

```json
{
  "nwp": "0.4",
  "node_id": "urn:nps:node:api.example.com:orders",
  "node_type": "complex",
  "display_name": "Order Management Node",
  "manifest_version": "etag-2026041402",
  "min_agent_version": "0.3",
  "wire_formats": ["ncp-capsule", "msgpack", "json"],
  "preferred_format": "ncp-capsule",
  "schema_anchors": {
    "order":   "sha256:a3f9b2c1...",
    "product": "sha256:b2c1d3e4..."
  },
  "capabilities": {
    "query": true,
    "stream_query": true,
    "aggregate": true,
    "subscribe": true,
    "subscribe_filter": true,
    "vector_search": false,
    "token_budget_hint": true,
    "ext_frame": false,
    "e2e_enc": false,
    "inline_anchor": true
  },
  "data_sources": ["rds:orders_db"],
  "auth": {
    "required": true,
    "identity_type": "nip-cert",
    "trusted_issuers": ["https://ca.mycompany.com"],
    "required_capabilities": ["nwp:query", "nwp:invoke"],
    "scope_check": "prefix"
  },
  "rate_limits": {
    "requests_per_minute": 300,
    "requests_per_day": 50000,
    "max_concurrent_streams": 10,
    "max_subscriptions": 5
  },
  "actions": {
    "orders.create": {
      "description": "Create a new order",
      "params_anchor": "sha256:create_params...",
      "result_anchor": "sha256:order...",
      "async": true,
      "idempotent": true,
      "timeout_ms_default": 10000,
      "timeout_ms_max": 60000,
      "required_capability": "nwp:invoke"
    },
    "orders.cancel": {
      "description": "Cancel an existing order",
      "params_anchor": "sha256:cancel_params...",
      "result_anchor": "sha256:cancel_result...",
      "async": false,
      "idempotent": true,
      "timeout_ms_default": 5000,
      "timeout_ms_max": 10000,
      "required_capability": "nwp:invoke"
    }
  },
  "endpoints": {
    "query":     "nwp://api.example.com/orders/query",
    "stream":    "nwp://api.example.com/orders/stream",
    "invoke":    "nwp://api.example.com/orders/invoke",
    "subscribe": "nwp://api.example.com/orders/subscribe",
    "actions":   "nwp://api.example.com/orders/actions",
    "schema":    "nwp://api.example.com/orders/.schema"
  },
  "tokenizer_support": ["cl100k_base", "claude"]
}
```

**NWM Conditional Requests**

Agents SHOULD cache the NWM and use `manifest_version` for conditional requests: in HTTP mode via the `If-None-Match: {manifest_version}` header; if the manifest is unchanged the server returns `304 Not Modified`.

### 4.6 NWM Action Registry

The `actions` field is an `{action_id: ActionSpec}` dictionary. Action/Complex/Gateway Nodes MUST declare all callable operations here.

**ActionSpec Field Definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | string | Optional | Human-readable description of the operation |
| `params_anchor` | string | Optional | anchor_id of the parameter Schema (Agent uses this to validate ActionFrame.params) |
| `result_anchor` | string | Optional | anchor_id of the result Schema (CapsFrame uses this anchor_ref on success) |
| `async` | bool | Required | Whether async execution is supported (if true, ActionFrame may set async=true) |
| `idempotent` | bool | Optional | Whether the operation is idempotent (Agents may safely retry if true) |
| `timeout_ms_default` | uint32 | Optional | Default timeout in milliseconds |
| `timeout_ms_max` | uint32 | Optional | Maximum allowed timeout in milliseconds |
| `required_capability` | string | Optional | NIP capability required to invoke this operation, e.g. `"nwp:invoke"` |
| `min_assurance_level` | string | Optional | Per-action assurance-level override: `"anonymous"` / `"attested"` / `"verified"`. When present, takes precedence over the top-level NWM `min_assurance_level` for requests targeting this action. Requests presenting a lower level MUST be rejected with `NWP-AUTH-ASSURANCE-TOO-LOW`. (NPS-RFC-0003) |

**`/actions` Endpoint**

An Agent sends `GET /actions`; the node returns the full NWM `actions` field as JSON (for dynamic discovery without downloading the entire NWM):

```json
{
  "node_id": "urn:nps:node:api.example.com:orders",
  "actions": {
    "orders.create": { ... },
    "orders.cancel": { ... }
  }
}
```

---

## 5. Schema Retrieval Flow

Agents retrieve a Node's schema via the following flow (AnchorFrames are published by Nodes; Agents are read-only consumers):

```
Agent                              Node
  │                                  │
  │── GET /.nwm ─────────────────→   │  1. Read manifest, get schema_anchors
  │←── NWM JSON ──────────────────   │     { "order": "sha256:a3f9..." }
  │                                  │
  │── GET /.schema ──────────────→   │  2. Fetch complete AnchorFrame (on demand)
  │←── AnchorFrame JSON ──────────   │     Agent caches locally
  │                                  │
  │── QueryFrame(anchor_ref) ────→   │  3. Query carries only anchor_ref
  │←── CapsFrame(anchor_ref) ─────   │
```

Agents SHOULD preload AnchorFrames for all `schema_anchors` declared in the NWM on first connection, reducing latency for subsequent requests.

---

## 6. QueryFrame (0x10)

Used for structured data queries on Memory Nodes.

### 6.1 Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x10` |
| `type` | string | Optional | Reserved query type identifier per §12. When set, type-specific fields apply and `anchor_ref` semantics are defined by the type. Absent: per-anchor query behavior below |
| `anchor_ref` | string | Conditionally Required | Schema anchor_id; may be omitted for aggregation queries or when `type` selects a reserved type that does not require it |
| `auto_anchor` | bool | Optional | If true and the anchor is stale, the Node automatically attaches the latest AnchorFrame in the response. Default: true |
| `stream` | bool | Optional | If true, triggers streaming query mode (see §6.6); response is a StreamFrame sequence rather than a CapsFrame |
| `aggregate` | object | Optional | Aggregation operation (see §6.7); when set, `anchor_ref` may be omitted |
| `filter` | object | Optional | Filter conditions, see §6.2 |
| `fields` | array | Optional | List of fields to return; omit to return all fields |
| `limit` | uint32 | Optional | Maximum records to return, default 20, max 1000; for streaming queries, max records per frame |
| `cursor` | string | Optional | Pagination cursor from the previous response's `next_cursor` |
| `order` | array | Optional | Sort rules, see §6.3 |
| `vector_search` | object | Optional | Vector similarity search, see §6.4 |
| `token_budget` | uint32 | Optional | CGN budget limit (native mode equivalent of `X-NWP-Budget`) |
| `tokenizer` | string | Optional | Tokenizer identifier in use (native mode equivalent of `X-NWP-Tokenizer`) |
| `depth` | uint8 | Optional | Node graph traversal depth, default 1, max 5 (native mode equivalent of `X-NWP-Depth`) |
| `request_id` | string | Optional | UUID v4 for request tracing; echoed back by the node in response and logs |

### 6.2 Filter Syntax

| Operator | Meaning | Example |
|----------|---------|---------|
| `$eq` | Equal | `{ "status": { "$eq": "active" } }` |
| `$ne` | Not equal | `{ "status": { "$ne": "deleted" } }` |
| `$lt` | Less than | `{ "price": { "$lt": 500 } }` |
| `$lte` | Less than or equal | `{ "price": { "$lte": 500 } }` |
| `$gt` | Greater than | `{ "stock": { "$gt": 0 } }` |
| `$gte` | Greater than or equal | `{ "rating": { "$gte": 4.0 } }` |
| `$in` | In list | `{ "category": { "$in": ["phone", "tablet"] } }` |
| `$nin` | Not in list | `{ "tag": { "$nin": ["discontinued"] } }` |
| `$contains` | String contains (case-sensitive) | `{ "name": { "$contains": "Pro" } }` |
| `$between` | Range (inclusive on both ends) | `{ "price": { "$between": [100, 500] } }` |
| `$exists` | Field exists check | `{ "thumbnail": { "$exists": true } }` |
| `$regex` | Regex match (UTF-8) | `{ "sku": { "$regex": "^PROD-[0-9]{4}$" } }` |
| `$and` | Logical AND | `{ "$and": [ {...}, {...} ] }` |
| `$or` | Logical OR | `{ "$or": [ {...}, {...} ] }` |
| `$not` | Logical NOT | `{ "$not": { "status": { "$eq": "deleted" } } }` |

**`$regex` Security Constraints**: Pattern length ≤ 256 characters; nested quantifiers (e.g. `(a+)+`) are prohibited; nodes MUST perform ReDoS detection and return `NWP-QUERY-REGEX-UNSAFE` on violation.

Filter nesting depth MUST be ≤ 8 levels.

### 6.3 Sort Rules

```json
{ "order": [{ "field": "price", "dir": "ASC" }, { "field": "name", "dir": "ASC" }] }
```

### 6.4 Vector Search

```json
{
  "vector_search": {
    "field": "embedding",
    "vector": [0.12, -0.34, 0.56],
    "top_k": 10,
    "threshold": 0.85,
    "metric": "cosine"
  }
}
```

Supported `metric` values: `cosine` (default), `euclidean`, `dot_product`. Nodes declare support via `capabilities.vector_search=true`; unsupported nodes return `NWP-QUERY-VECTOR-UNSUPPORTED`.

### 6.5 Single Query Complete Example

```json
{
  "frame": "0x10",
  "anchor_ref": "sha256:a3f9b2c1...",
  "auto_anchor": true,
  "filter": {
    "$and": [
      { "category": { "$eq": "electronics" } },
      { "price": { "$lt": 500 } },
      { "stock": { "$gt": 0 } }
    ]
  },
  "fields": ["id", "name", "price", "stock"],
  "limit": 20,
  "order": [{ "field": "price", "dir": "ASC" }],
  "token_budget": 800,
  "tokenizer": "cl100k_base",
  "request_id": "550e8400-e29b-41d4-a716-446655440001"
}
```

### 6.6 Streaming Query Protocol

When QueryFrame contains `stream: true` (or uses the `/stream` sub-path), the node responds with a **StreamFrame (0x03) sequence** rather than a single CapsFrame. Requires `capabilities.stream_query=true`.

**Streaming Query Flow**

```
Agent                              Node
  │                                  │
  │── QueryFrame(stream:true) ────→  │
  │                                  │  Query in batches, limit records per batch
  │  ←── StreamFrame(seq=0) ───────  │  First frame, contains anchor_ref and estimated_total
  │  ←── StreamFrame(seq=1) ───────  │  Subsequent frames, data is the next batch of records
  │       ...                        │
  │  ←── StreamFrame(is_last=true) ─ │  Final frame, is_last=true, data may be empty
```

**First Frame Additional Fields (StreamFrame Extension)**

For streaming queries, the first frame (seq=0) SHOULD include metadata:

| Field | Type | Description |
|-------|------|-------------|
| `estimated_total` | uint64 | Estimated total records matching the filter; -1 means unknown |
| `request_id` | string | Echo of the QueryFrame's request_id |

**Pagination vs. Streaming**

- Streaming queries do not use `cursor`; records are pushed continuously per `order` until `limit × frames` or full push completes
- To terminate early, the Agent sends a SubscribeFrame (`action="unsubscribe"`, `stream_id` equals QueryFrame's `request_id`) or disconnects. **Note**: reusing SubscribeFrame for streaming-query cancellation is intentional — it is the protocol-wide stream-cancellation signal; nodes route it by `stream_id` regardless of whether the stream originated from a QueryFrame or a SubscribeFrame.
- The node MUST stop pushing after the connection is closed

### 6.7 Aggregation Queries

When QueryFrame contains an `aggregate` field, the node returns aggregated results rather than raw records. Requires `capabilities.aggregate=true`.

**aggregate Field Definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `operations` | array | Required | List of aggregation operations, see below |
| `group_by` | array | Optional | Grouping field list, e.g. `["category", "status"]` |
| `having` | object | Optional | Post-grouping filter (same syntax as filter, but field names are aliases) |

**operation Element**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `func` | string | Required | `COUNT` / `SUM` / `AVG` / `MIN` / `MAX` / `COUNT_DISTINCT` |
| `field` | string | Conditionally Required | Field to aggregate (`COUNT` may omit, meaning row count) |
| `alias` | string | Required | Result field name |

**Aggregation Query Example**

```json
{
  "frame": "0x10",
  "filter": { "status": { "$eq": "active" } },
  "aggregate": {
    "operations": [
      { "func": "COUNT", "alias": "total" },
      { "func": "SUM",   "field": "price",  "alias": "revenue" },
      { "func": "AVG",   "field": "rating", "alias": "avg_rating" }
    ],
    "group_by": ["category"],
    "having": { "total": { "$gt": 10 } }
  },
  "order": [{ "field": "revenue", "dir": "DESC" }],
  "request_id": "550e8400-e29b-41d4-a716-446655440002"
}
```

**Aggregation Response (CapsFrame)**

Aggregation responses do not use a business schema; `anchor_ref` is fixed as `"nps:system:aggregate:result"`:

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:aggregate:result",
  "count": 3,
  "data": [
    { "category": "electronics", "total": 142, "revenue": 89450.00, "avg_rating": 4.3 },
    { "category": "clothing",    "total": 87,  "revenue": 12300.00, "avg_rating": 4.1 },
    { "category": "books",       "total": 56,  "revenue": 3200.00,  "avg_rating": 4.6 }
  ]
}
```

---

## 7. ActionFrame (0x11)

Used for operation invocation on Action Nodes and Complex Nodes.

### 7.1 Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x11` |
| `action_id` | string | Required | Operation identifier, format: `{domain}.{verb}`; system-reserved operations see §7.3 |
| `params` | object | Optional | Operation parameters; schema defined by NWM `actions.{action_id}.params_anchor` |
| `idempotency_key` | string | Optional | Idempotency key (UUID v4), valid for 24 hours |
| `timeout_ms` | uint32 | Optional | Timeout in milliseconds, default 5000, max 300000 |
| `async` | bool | Optional | If true, execute asynchronously; response returns `task_id` |
| `callback_url` | string | Optional | Callback URL for async task completion (`https://` only) |
| `priority` | string | Optional | Task priority: `"low"` / `"normal"` (default) / `"high"` |
| `request_id` | string | Optional | UUID v4 for request tracing (echoed in response and task status) |

### 7.2 Async Task State Machine

```
PENDING → RUNNING → COMPLETED
                  ↘ FAILED
                  ↘ CANCELLED
```

For async execution, the initial response (CapsFrame):

```json
{
  "task_id": "uuid-v4",
  "status": "pending",
  "poll_url": "nwp://api.example.com/orders/actions/status/uuid-v4",
  "estimated_ms": 3000,
  "request_id": "550e8400-..."
}
```

### 7.3 System-Reserved Operations

All nodes supporting async Actions MUST implement:

| action_id | Description | Required Params | Response |
|-----------|-------------|-----------------|----------|
| `system.task.status` | Poll task status | `{ "task_id": "uuid" }` | Task status object (see below) |
| `system.task.cancel` | Cancel a task | `{ "task_id": "uuid" }` | `{ "cancelled": true }` or error |

**`system.task.status` Response**

```json
{
  "task_id": "uuid-v4",
  "status": "running",
  "progress": 0.42,
  "created_at": "2026-04-14T10:00:00Z",
  "updated_at": "2026-04-14T10:00:05Z",
  "request_id": "550e8400-...",
  "result": null,
  "error": null
}
```

### 7.4 Complete Example

```json
{
  "frame": "0x11",
  "action_id": "orders.create",
  "params": { "product_id": 1001, "quantity": 2 },
  "idempotency_key": "550e8400-e29b-41d4-a716-446655440000",
  "timeout_ms": 10000,
  "async": true,
  "callback_url": "https://agent.myapp.com/callbacks/nwp",
  "priority": "normal",
  "request_id": "550e8400-e29b-41d4-a716-446655440003"
}
```

---

## 8. SubscribeFrame (0x12)

Used to establish change subscriptions on Memory Nodes. The server pushes incremental updates via DiffFrame (0x02).

### 8.1 Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x12` |
| `action` | string | Required | `"subscribe"` / `"unsubscribe"` / `"ping"` |
| `type` | string | Optional | Reserved subscribe type identifier per §12. When set, type-specific fields apply and `anchor_ref` semantics are defined by the type. Absent: per-anchor subscribe behavior below |
| `stream_id` | string | Required | Client-generated UUID v4 |
| `anchor_ref` | string | Conditionally Required | anchor_id of the subscribed data (required when `action="subscribe"` and `type` is absent or the reserved type requires an anchor) |
| `filter` | object | Optional | Filter conditions (same as §6.2); requires `capabilities.subscribe_filter=true` |
| `heartbeat_interval` | uint32 | Optional | Heartbeat interval in seconds, 0 = disabled, default 30 |
| `resume_from_seq` | uint64 | Optional | Provided on reconnection; replays events after this sequence number (inclusive); node returns `NWP-SUBSCRIBE-SEQ-TOO-OLD` if it cannot satisfy the request |

### 8.2 DiffFrame Subscription Extension Fields

Subscription-pushed DiffFrames (0x02) add the following to the standard fields:

| Field | Type | Description |
|-------|------|-------------|
| `stream_id` | string | Associated subscription stream ID |
| `seq` | uint64 | Monotonically increasing event sequence number (per-stream, starting from 1); Agents use this to detect dropped frames and support reconnection |
| `event_type` | string | `"create"` / `"update"` / `"delete"` for default subscriptions. Reserved subscribe types (§12) MAY define additional values (e.g. topology event types per §12.2) |
| `timestamp` | string | Time of change (ISO 8601) |

**Sequence Number Semantics**

- Each `stream_id` maintains its own independent `seq`, starting from 1 and monotonically increasing
- When the Agent detects a `seq` gap, it SHOULD re-subscribe using `resume_from_seq`
- Nodes SHOULD buffer recent N events (recommended: 10 minutes or 10,000 events, whichever comes first)
- If `resume_from_seq` is outside the buffer range, the node MUST return `NWP-SUBSCRIBE-SEQ-TOO-OLD` (Agent must do a full re-query before re-subscribing)

### 8.3 Subscription Flow

```
Agent                                  Node
  │                                       │
  │── SubscribeFrame(subscribe, sid) ───→ │  Establish subscription
  │  ←─────────────────── CapsFrame(ack) │  Confirm (stream_id, status="subscribed")
  │                                       │
  │  ←─── DiffFrame(seq=1, event) ──────  │  Change push
  │  ←─── DiffFrame(seq=2, event) ──────  │
  │  ←─── DiffFrame(seq=N, ping) ───────  │  Heartbeat (patch=[])
  │                                       │
  │  [ Connection interrupted ]           │
  │── SubscribeFrame(subscribe, sid,      │  Reconnection with resume
  │      resume_from_seq=N) ───────────→  │
  │  ←─────────────────── CapsFrame(ack) │  Confirm (status="subscribed", resumed=true)
  │  ←─── DiffFrame(seq=N+1, ...) ──────  │  Replay from breakpoint
```

**Subscription Acknowledgement CapsFrame**

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:subscribe:ack",
  "count": 1,
  "data": [{
    "stream_id": "550e8400-...",
    "status": "subscribed",
    "resumed": false,
    "last_seq": 0,
    "effective_filter": {}
  }]
}
```

- `last_seq`: The latest event seq the node currently knows (0 for new subscriptions, or the most recent buffered event's seq for resumptions)
- `resumed`: true indicates this is a reconnection resumption

### 8.4 Subscription in HTTP Mode (SSE)

```
POST /nwp/orders/subscribe HTTP/1.1
Content-Type: application/nwp-frame

[SubscribeFrame bytes]

HTTP/1.1 200 OK
Content-Type: text/event-stream

data: [DiffFrame bytes, base64]

data: [DiffFrame bytes, base64]
```

---

## 9. HTTP Headers (HTTP Mode)

### 9.1 Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-NWP-Agent` | Conditionally Required | Agent NID (required when auth.required=true) |
| `X-NWP-Budget` | Optional | CGN budget limit (uint32) |
| `X-NWP-Tokenizer` | Optional | Tokenizer identifier used by the Agent |
| `X-NWP-Depth` | Optional | Node graph traversal depth, default 1, max 5 |
| `X-NWP-Encoding` | Optional | Request encoding tier: `json`/`msgpack`, default `msgpack` |
| `X-NWP-Request-ID` | Optional | UUID v4 request tracing ID; echoed back by the node in the response header |
| `If-None-Match` | Optional | NWM conditional request; value is `manifest_version` |
| `Content-Type` | Required | `application/nwp-frame` |

### 9.2 Response Headers

| Header | Description |
|--------|-------------|
| `X-NWP-Schema` | anchor_id used in the response |
| `X-NWP-Tokens` | Actual CGN consumed |
| `X-NWP-Tokens-Native` | Native token consumption |
| `X-NWP-Tokenizer-Used` | Tokenizer actually used |
| `X-NWP-Cached` | `true` indicates a cache hit |
| `X-NWP-Node-Type` | Node type |
| `X-NWP-Request-ID` | Echo of the requester's `X-NWP-Request-ID` (node MAY auto-generate if not provided) |
| `X-NWP-Rate-Limit` | Max requests per minute |
| `X-NWP-Rate-Remaining` | Remaining requests this minute |
| `X-NWP-Rate-Reset` | Rate limit window reset time (Unix timestamp) |
| `Content-Type` | `application/nwp-capsule` (normal response) / `application/nwp-error+json` (error response) |

### 9.3 Native Mode Field Mapping

| HTTP Header | QueryFrame Field | ActionFrame Field |
|-------------|-----------------|-------------------|
| `X-NWP-Agent` | — (declared in HelloFrame `agent_id` handshake) | Same |
| `X-NWP-Budget` | `token_budget` | — |
| `X-NWP-Tokenizer` | `tokenizer` | — |
| `X-NWP-Depth` | `depth` | — |
| `X-NWP-Request-ID` | `request_id` | `request_id` |

### 9.4 HTTP Mode Error Response Format

In HTTP mode, error responses use the following format, `Content-Type: application/nwp-error+json`:

```json
{
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NWP-ACTION-NOT-FOUND",
  "message": "Action 'orders.ship' is not registered on this node",
  "details": { "action_id": "orders.ship" },
  "request_id": "550e8400-e29b-41d4-a716-446655440003"
}
```

The HTTP status code is determined by the NPS status code mapping; see [status-codes.md](status-codes.md).

**Field Descriptions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | Required | NPS status code |
| `error` | string | Required | Protocol-level error code (e.g. `NWP-ACTION-NOT-FOUND`) |
| `message` | string | Optional | Human-readable description |
| `details` | object | Optional | Structured additional error information |
| `request_id` | string | Optional | Echo of the `X-NWP-Request-ID` from the request |

---

## 10. Complete Request/Response Example

**HTTP Mode Query Request**

```
POST /nwp/orders/query HTTP/1.1
Host: api.example.com:17433
X-NWP-Agent: urn:nps:agent:ca.innolotus.com:550e8400
X-NWP-Budget: 1200
X-NWP-Tokenizer: cl100k_base
X-NWP-Request-ID: 550e8400-e29b-41d4-a716-446655440001
Content-Type: application/nwp-frame

[QueryFrame: { anchor_ref, filter, fields, limit, auto_anchor }]
```

**Success Response**

```
HTTP/1.1 200 OK
X-NWP-Schema: sha256:a3f9...
X-NWP-Tokens: 380
X-NWP-Request-ID: 550e8400-e29b-41d4-a716-446655440001
X-NWP-Rate-Limit: 300
X-NWP-Rate-Remaining: 248
Content-Type: application/nwp-capsule

[CapsFrame]
```

**Error Response**

```
HTTP/1.1 404 Not Found
X-NWP-Request-ID: 550e8400-e29b-41d4-a716-446655440001
Content-Type: application/nwp-error+json

{ "status": "NPS-CLIENT-NOT-FOUND", "error": "NWP-QUERY-FIELD-UNKNOWN", ... }
```

---

## 11. Complex Node — Node Graph

Complex Nodes declare sub-node references in the NWM:

```json
{
  "graph": {
    "refs": [
      { "rel": "user",    "node": "nwp://api.myapp.com/users" },
      { "rel": "payment", "node": "nwp://pay.myapp.com/transactions" }
    ],
    "max_depth": 2
  }
}
```

Agents control traversal depth via the `X-NWP-Depth` header (HTTP mode) or the QueryFrame `depth` field (native mode). Nodes MUST detect circular references (return `NWP-GRAPH-CYCLE`) and maintain a sub-node URL whitelist (SSRF prevention).

---

## 12. Reserved Query Types

The `type` field on `QueryFrame` (§6.1) and `SubscribeFrame` (§8.1) opts a request into a **reserved query type** with specification-defined semantics. Identifiers in the `topology.*` namespace are reserved for cluster topology operations on Anchor Nodes; all reserved namespaces below are **mandatory** for the node roles indicated.

| Namespace | Owning role | Mandatory at | Operations |
|-----------|-------------|--------------|------------|
| `topology.*` | Anchor Node (§2.1) | NPS-AaaS Profile L2 ([services/NPS-AaaS-Profile.md §4.3](services/NPS-AaaS-Profile.md)) | `topology.snapshot` (§12.1), `topology.stream` (§12.2) |

When `type` is absent, default per-anchor query/subscribe semantics apply (§6, §8). When `type` is set, the per-type fields documented below apply and any conflicting standard fields (e.g. `anchor_ref`, top-level `filter`) are ignored unless the per-type schema explicitly carries them.

Implementations that do not recognize a reserved `type` value MUST reject the request with `NWP-RESERVED-TYPE-UNSUPPORTED` (§13) so the caller can distinguish "unknown reserved operation" from "action_id not found" (`NWP-ACTION-NOT-FOUND`) and fall back or fail explicitly.

### 12.1 `topology.snapshot`

Single-shot retrieval of an Anchor Node's cluster topology.

| Property | Value |
|----------|-------|
| Frame | QueryFrame (0x10) with `type = "topology.snapshot"` |
| Required of | All Anchor Nodes (mandatory at NPS-AaaS Profile L2 and above) |
| Idempotent | Yes |
| Caching | Responses MAY be cached client-side; the `version` field correlates the snapshot with subsequent `topology.stream` events |

**Request fields** (top-level on QueryFrame, supplementing §6.1):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Required | Constant `"topology.snapshot"` |
| `topology` | object | Required | Container for topology-specific parameters per below |
| `topology.scope` | string | Required | `"cluster"` for the Anchor's own cluster; `"member"` for a single member (requires `topology.target_nid`) |
| `topology.include` | array of strings | Optional | Subset of `["members", "capabilities", "tags", "metrics"]`. Default `["members"]`. The `capabilities` and `metrics` schemas are implementation-defined and MAY be empty |
| `topology.depth` | uint8 | Optional | For sub-Anchor members, controls recursion. `1` (default) lists sub-Anchors as references only; `2+` recurses. Anchor Nodes MAY cap depth and set `truncated: true` when exceeded. Sub-Anchor recursion at depth ≥ 2 is **OPTIONAL** at L2; clients SHOULD recurse manually by issuing one snapshot per sub-Anchor |
| `topology.target_nid` | string | Conditionally Required | Required when `topology.scope = "member"`; the NID of the member to introspect |

**Response**: `CapsFrame (0x04)` with `anchor_ref = "nps:system:topology:snapshot"` and a single-element `data` array carrying the snapshot payload below.

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:topology:snapshot",
  "count": 1,
  "data": [{
    "version": 142,
    "anchor_nid": "urn:nps:node:labacacia:host-abc123",
    "cluster_size": 23,
    "members": [
      {
        "nid": "urn:nps:agent:labacacia:host-abc123-sess-aaa",
        "node_roles": ["memory"],
        "activation_mode": "ephemeral",
        "tags": ["dev", "library"],
        "joined_at": "2026-04-15T10:23:00Z",
        "last_seen": "2026-04-26T14:55:00Z"
      },
      {
        "nid": "urn:nps:node:labacacia:host-def456",
        "node_roles": ["anchor"],
        "activation_mode": "resident",
        "child_anchor": true,
        "member_count": 7,
        "tags": ["sub-cluster", "training"]
      }
    ],
    "truncated": false
  }]
}
```

**Snapshot payload fields**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | uint64 | Required | Monotonically increasing topology version. Resets only on Anchor restart / rebase (§12.3) |
| `anchor_nid` | string | Required | NID of the responding Anchor Node |
| `cluster_size` | uint32 | Required | Total direct members, regardless of `depth` truncation |
| `members` | array of member objects | Required | Per the member object schema below |
| `truncated` | bool | Optional | True iff `topology.depth` cap was hit; otherwise omitted or false |

**Member object schema**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `nid` | string | Required | Member NID |
| `node_roles` | array of strings | Required | NDP `node_roles` — the full role set declared by this member (NPS-4 §3.1) |
| `activation_mode` | string | Required | NDP `activation_mode` — one of `ephemeral` / `resident` / `hybrid` (NPS-4) |
| `child_anchor` | bool | Optional | True if this member is itself an Anchor Node of a sub-cluster; implies `member_count` |
| `member_count` | uint32 | Conditionally Required | Required when `child_anchor = true`; count of the sub-Anchor's direct members |
| `tags` | array of strings | Optional | NDP-declared tags |
| `joined_at` | string | Optional | RFC 3339 timestamp; first observed |
| `last_seen` | string | Optional | RFC 3339 timestamp; most recent NDP `Announce` |
| `capabilities` | object | Optional | Returned only if requested via `topology.include`; schema implementation-defined |
| `metrics` | object | Optional | Returned only if requested via `topology.include`; schema implementation-defined |

### 12.2 `topology.stream`

Continuous topology change feed for an Anchor Node's cluster.

| Property | Value |
|----------|-------|
| Frame | SubscribeFrame (0x12) with `type = "topology.stream"` |
| Required of | All Anchor Nodes (mandatory at NPS-AaaS Profile L2 and above) |
| Cancelable | Yes — via standard `SubscribeFrame.action = "unsubscribe"` |

**Request fields** (top-level on SubscribeFrame, supplementing §8.1):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Required | Constant `"topology.stream"` |
| `topology` | object | Required | Container for topology-specific parameters per below |
| `topology.scope` | string | Required | `"cluster"` (default for Anchor's own cluster); future scopes reserved |
| `topology.filter` | object | Optional | Reduces event volume. Supported keys: `tags_any` (array, match-any), `tags_all` (array, match-all), `node_roles` (array — filter by role, matches members whose `node_roles` intersects the given values). Anchor Nodes MUST reject unsupported filter keys with `NWP-TOPOLOGY-FILTER-UNSUPPORTED` |
| `topology.since_version` | uint64 | Optional | Resume from a previous version. Anchor Node MUST replay missed events when feasible; if the version is outside the retention window, MUST emit a `resync_required` event and the client MUST issue a fresh `topology.snapshot` |

For `type = "topology.stream"`, `topology.since_version` is the topology-scoped synonym of `SubscribeFrame.resume_from_seq` (§8.1). When both fields are present, `topology.since_version` takes precedence.

**Events** are pushed as `DiffFrame (0x02)` with the §8.2 extension fields. For `topology.stream` subscriptions, `event_type` carries one of the topology event types below (extending the default `"create" / "update" / "delete"` enum), `seq` is the post-event topology version (§12.3), and `payload` carries the event-specific data.

| `event_type` | Trigger | `payload` shape |
|--------------|---------|-----------------|
| `member_joined` | NDP `Announce` from a node naming this Anchor as `cluster_anchor` | Full member object (§12.1) |
| `member_left` | Member explicitly leaves OR exceeds NDP liveness TTL | `{ "nid": "urn:nps:..." }` |
| `member_updated` | Existing member's metadata changes (tags, activation_mode, capabilities, etc.) | `{ "nid": "urn:nps:...", "changes": { "<field>": <new value>, ... } }` — field-level diff only; reassembly is the client's responsibility |
| `anchor_state` | Rare. Anchor Node internal state change relevant to clients (e.g., version counter rebase after restart) | `{ "field": "version_rebased", "details": { ... } }` |
| `resync_required` | Subscriber's `topology.since_version` is no longer replayable | `{ "reason": "version_too_old" }`. This event omits `seq` and the subscriber MUST issue a fresh `topology.snapshot` |

Standard SubscribeFrame heartbeats (§8.2) and unsubscribe (§8.1, `action = "unsubscribe"`) operate unchanged.

### 12.3 Versioning and Consistency Model

**Guaranteed**:

- A `topology.snapshot` returned at `version: V` reflects the cluster state after exactly `V` topology mutations.
- A `topology.stream` event with `seq: V` reflects the cluster state after exactly `V` mutations.
- A snapshot at `V` combined with all subsequent stream events `V+1, V+2, …` yields a consistent live view.

**Not guaranteed**:

- Real-time delivery latency — events MAY batch.
- Event ordering across multiple Anchor Nodes — each Anchor maintains its own `version` counter.
- Total ordering with non-topology events (Action / Query traffic on member nodes).

**Restart and rebase**: An Anchor Node MAY rebase its `version` counter on restart. When rebasing, the Anchor MUST emit an `anchor_state` event with `field: "version_rebased"` to all active subscribers; subscribers MUST treat this as equivalent to `resync_required` and issue a fresh `topology.snapshot`.

### 12.4 Out of Scope

- **Capability and metric schema standardization**: §12.1's `capabilities` and `metrics` field schemas are implementation-defined; standardization is deferred to a follow-up CR once enough implementations exist.
- **Cross-cluster federation queries**: Querying topology across multiple Anchor Nodes is an NPS-AaaS Profile L3 / NPS Cloud concern. This section is single-Anchor only.
- **Authorization model — minimum binding (Phase 1–2)**: Anchor Nodes MUST enforce the following minimum before serving any `topology.*` request:

  1. **Capability gate**: The requesting NID MUST declare `topology:read` in `IdentFrame.capabilities` (NPS-3 §5.1); absent capability MUST produce `NWP-TOPOLOGY-UNAUTHORIZED`. The IdentFrame is signed by the requester's private key, so the claim is integrity-protected but self-declared — it is not CA-attested at Phase 1–2.
  2. **NDP role cross-check (defense-in-depth)**: The Anchor SHOULD additionally verify that the requester's last received `AnnounceFrame` (within TTL) declares `node_roles` containing `"anchor"`. A mismatch SHOULD produce `NWP-TOPOLOGY-UNAUTHORIZED` with a `hint`. An absent `AnnounceFrame` MUST NOT block a requester that has passed the capability gate.
  3. **Phase 3 [RFC-0002 stable]**: Anchors SHOULD additionally verify a CA-attested `id-nps-node-roles` cert extension (to be defined in a follow-up RFC-0002 amendment) to close the self-declaration gap and bind the role claim to a CA-issued certificate.

  Fine-grained per-cluster namespace or ACL policies remain implementation-defined and are tracked for a follow-up CR.
- **Browser transport (WebSocket)**: Whether `npsd` exposes a WebSocket endpoint for browser clients is tracked separately. Topology query semantics here are transport-independent.

---

## 13. Error Codes

| Error Code | NPS Status Code | Description |
|-----------|----------------|-------------|
| `NWP-AUTH-NID-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Agent scope does not cover the target node |
| `NWP-AUTH-NID-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | NID certificate has expired |
| `NWP-AUTH-NID-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | NID has been revoked |
| `NWP-AUTH-NID-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | NID issuer not in trusted_issuers |
| `NWP-AUTH-NID-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Agent lacks a capability required by the node |
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` | Agent's `assurance_level` is below the node's `min_assurance_level` (or the per-action override). Response SHOULD include a `hint` pointing to a CA enrolment URL. (NPS-RFC-0003) |
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` | The receiving Node's `reputation_policy` (Phase 2 NWM field — see [NPS-RFC-0004 §4.4](rfcs/NPS-RFC-0004-nid-reputation-log.md)) matched a `reject_on` rule against the requesting `subject_nid`. Response SHOULD include the matching `incident` + `severity` + log entry `seq` for traceability. The field shape that produces this error lands at NWP v0.8 (Phase 2); the error code itself is reserved at NWP v0.7 (Phase 1) so SDKs can begin recognising it without round-tripping through a future spec bump. (NPS-RFC-0004) |
| `NWP-QUERY-FILTER-INVALID` | `NPS-CLIENT-BAD-PARAM` | Filter syntax is invalid or nesting is too deep |
| `NWP-QUERY-FIELD-UNKNOWN` | `NPS-CLIENT-BAD-PARAM` | fields references a non-existent field |
| `NWP-QUERY-CURSOR-INVALID` | `NPS-CLIENT-BAD-PARAM` | cursor value cannot be decoded or has expired |
| `NWP-QUERY-REGEX-UNSAFE` | `NPS-CLIENT-BAD-PARAM` | `$regex` pattern rejected (ReDoS risk or too long) |
| `NWP-QUERY-VECTOR-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support vector search |
| `NWP-QUERY-AGGREGATE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support aggregation queries |
| `NWP-QUERY-AGGREGATE-INVALID` | `NPS-CLIENT-BAD-PARAM` | aggregate structure is invalid (unknown func, duplicate alias, etc.) |
| `NWP-QUERY-STREAM-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support streaming queries |
| `NWP-ACTION-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | action_id does not exist |
| `NWP-ACTION-PARAMS-INVALID` | `NPS-CLIENT-UNPROCESSABLE` | Operation parameter schema validation failed |
| `NWP-ACTION-IDEMPOTENCY-CONFLICT` | `NPS-CLIENT-CONFLICT` | A request with the same idempotency_key is already in progress |
| `NWP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | task_id does not exist |
| `NWP-TASK-ALREADY-CANCELLED` | `NPS-CLIENT-CONFLICT` | Task has already been cancelled |
| `NWP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | Task has already completed; cannot cancel |
| `NWP-TASK-ALREADY-FAILED` | `NPS-CLIENT-CONFLICT` | Task has already failed; cannot cancel |
| `NWP-SUBSCRIBE-STREAM-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | stream_id referenced in unsubscribe does not exist |
| `NWP-SUBSCRIBE-LIMIT-EXCEEDED` | `NPS-LIMIT-EXCEEDED` | Exceeded maximum concurrent subscriptions |
| `NWP-SUBSCRIBE-FILTER-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support filtered subscriptions |
| `NWP-SUBSCRIBE-INTERRUPTED` | `NPS-SERVER-UNAVAILABLE` | Subscription stream terminated due to underlying data source interruption |
| `NWP-SUBSCRIBE-SEQ-TOO-OLD` | `NPS-CLIENT-CONFLICT` | resume_from_seq is outside the node's buffer range; full re-query required |
| `NWP-BUDGET-EXCEEDED` | `NPS-LIMIT-BUDGET` | Response would exceed the token budget |
| `NWP-DEPTH-EXCEEDED` | `NPS-CLIENT-BAD-PARAM` | depth exceeds the node's allowed max_depth |
| `NWP-GRAPH-CYCLE` | `NPS-CLIENT-UNPROCESSABLE` | Node graph contains a circular reference |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | Underlying data source temporarily unavailable |
| `NWP-MANIFEST-VERSION-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Agent NPS version is below the node's min_agent_version |
| `NWP-MANIFEST-NODE-TYPE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | NWM `node_type` contains the removed legacy value `"gateway"` (NPS-CR-0001); response SHOULD include a `hint` pointing to NPS-CR-0001 |
| `NWP-MANIFEST-NODE-TYPE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | NWM `node_type` contains an unrecognized value (see `NWP-MANIFEST-NODE-TYPE-REMOVED` for the `"gateway"` case) |
| `NWP-RATE-LIMIT-EXCEEDED` | `NPS-LIMIT-RATE` | Rate limit exceeded |
| `NWP-RESERVED-TYPE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | `QueryFrame` or `SubscribeFrame` `type` is an unrecognized reserved-type identifier (§12). Distinct from `NWP-ACTION-NOT-FOUND` — the unknown operand is `type`, not `action_id`. |
| `NWP-TOPOLOGY-UNAUTHORIZED` | `NPS-AUTH-FORBIDDEN` | Caller lacks permission to read this Anchor's topology (§12). Authorization policy is implementation-defined per §12.4 |
| `NWP-TOPOLOGY-UNSUPPORTED-SCOPE` | `NPS-CLIENT-BAD-PARAM` | `topology.scope` value is not implemented by this Anchor |
| `NWP-TOPOLOGY-DEPTH-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Requested `topology.depth` exceeds this Anchor's maximum |
| `NWP-TOPOLOGY-FILTER-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | `topology.filter` contains an unrecognized key |

---

## 14. Security Considerations

### 14.1 Scope Enforcement
Nodes MUST validate the Agent NID's scope on every request. Requests exceeding scope MUST return `NWP-AUTH-NID-SCOPE-VIOLATION` and MUST NOT return any data.

### 14.2 SSRF Protection
When Complex Nodes resolve sub-node references, they MUST maintain an allowlist of permitted node URL prefixes and MUST NOT access internal network addresses (RFC 1918).

### 14.3 Token Budget Enforcement
When the budget is exceeded, nodes SHOULD prefer trimming response content (field reduction → summary → record truncation); if trimming is not possible, they MUST return `NWP-BUDGET-EXCEEDED` and MUST NOT silently truncate data. See [token-budget.md §4.3](token-budget.md).

### 14.4 Rate Limiting
Nodes SHOULD enforce per-Agent NID rate limiting. On limit exceeded, return `NWP-RATE-LIMIT-EXCEEDED` with an `X-NWP-Rate-Reset` header. Unauthenticated requests SHOULD be limited by IP.

### 14.5 Filter Injection Protection
- Field names MUST contain only letters/digits/underscores/dots, length ≤ 128 characters
- `$regex` patterns MUST undergo ReDoS detection; filter nesting depth ≤ 8
- Nodes MUST use parameterized queries; string concatenation is prohibited

### 14.6 callback_url Abuse Prevention
- ActionFrame `callback_url` MUST use the `https://` scheme
- Nodes SHOULD perform SSRF checks on callback URLs (internal network addresses prohibited)
- Nodes SHOULD apply exponential backoff retries for failed callback deliveries (max 3 attempts), then abandon and mark the task as `COMPLETED` rather than retrying indefinitely

### 14.7 Topology Read-back
Anchor Nodes implementing §12 MUST treat `topology.snapshot` and `topology.stream` as authenticated surfaces. The minimum authorization binding is defined in §12.4: at Phase 1–2, the requesting NID MUST declare `topology:read` in `IdentFrame.capabilities` (primary gate); NDP `node_roles` cross-check is a defense-in-depth SHOULD. Unauthorized callers MUST receive `NWP-TOPOLOGY-UNAUTHORIZED` rather than silent empty responses to prevent oracle attacks against cluster membership.

---

## 15. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.10 | 2026-05-01 | §12.4 authorization model replaced "implementation-defined" with a Phase 1–2 minimum binding: Anchor Nodes MUST require `topology:read` in `IdentFrame.capabilities` (capability gate, self-declared but signed); SHOULD cross-check NDP `node_roles` contains `"anchor"` as defense-in-depth; Phase 3 [RFC-0002 stable] adds CA-attested `id-nps-node-roles` cert extension. §14.7 updated to reference §12.4 defined minimum instead of the previous hedging "SHOULD restrict" language. `Depends-On` NIP bumped to v0.6 (defines `topology:read` capability). |
| 0.9 | 2026-05-01 | **Breaking rename (pre-1.0)**: Topology member object field `node_kind` renamed to `node_roles` (§12.1); topology stream filter key `node_kind` renamed to `node_roles` (§12.2). §2.1 updated: `node_kind` reference to `node_roles`. New §2.1 **Node Role Resolution** section: `node_roles` (NDP, discovery-layer, array) and `node_type` (NWM, service-layer, string) are distinct fields — `node_type` MUST be one of the values in `node_roles`; validators SHOULD verify against cached NDP data. §4.1 `node_type` description updated with the cross-protocol constraint and pointer to §2.1. §14.7 `node_kind` reference updated to `node_roles`. Depends-On NDP bumped to v0.6. Fixes M1 naming-disambiguation issue. |
| 0.8 | 2026-04-27 | New §12 **Reserved Query Types** introducing the `topology.*` namespace mandatory at NPS-AaaS Profile L2: `topology.snapshot` (QueryFrame, `type="topology.snapshot"`) and `topology.stream` (SubscribeFrame, `type="topology.stream"`). Both QueryFrame §6.1 and SubscribeFrame §8.1 gain an optional top-level `type` field for opting into reserved types. DiffFrame §8.2 `event_type` enum extended via reserved subscribe types — `topology.stream` adds `member_joined` / `member_left` / `member_updated` / `anchor_state` / `resync_required`. Five new error codes: `NWP-TOPOLOGY-UNAUTHORIZED`, `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`, `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`, `NWP-TOPOLOGY-FILTER-UNSUPPORTED` (table §13). New §14.7 Topology Read-back security section. Existing §12 Error Codes / §13 Security / §14 Changelog renumbered to §13 / §14 / §15 to accommodate the new section. See [NPS-CR-0002](cr/NPS-CR-0002-anchor-topology-queries.md). |
| 0.7 | 2026-04-26 | **Breaking.** §2.1 Node Types: `Gateway Node` removed; replaced by **Anchor Node** (cluster control plane + NOP routing — inherits the existing role) and **Bridge Node** (NPS↔non-NPS protocol translation — new). NWM `node_type` enum updated; legacy `"gateway"` MUST be rejected. Anchor Node detailed semantics (§2.1 inline) cover member dispatch + optional registry; Bridge Node semantics cover HTTP/gRPC/MCP/A2A target adapters. Depends-On upgraded to NDP v0.5 for the `node_kind` array form + `cluster_anchor` + `bridge_protocols` fields. See [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md). |
| 0.6 | 2026-04-25 | NWM gains optional top-level `min_assurance_level` field (`anonymous` / `attested` / `verified`), with `min_assurance_level` per-action override on individual ActionSpecs (§4.6). New error code `NWP-AUTH-ASSURANCE-TOO-LOW` (`NPS-AUTH-FORBIDDEN`). `Depends-On` upgraded to NCP v0.6 (NPS-RFC-0001) and NIP v0.4 (NPS-RFC-0003). See [NPS-RFC-0003](rfcs/NPS-RFC-0003-agent-identity-assurance-levels.md). |
| 0.4 | 2026-04-14 | §3.2 added `/actions` sub-path; §4.1 NWM added `actions` field; §4.2 capabilities added stream_query and aggregate; §4.6 NWM Action Registry (ActionSpec, params_anchor/result_anchor/async/idempotent); QueryFrame §6.1 added `stream`, `aggregate`, `request_id`; §6.6 Streaming Query Protocol (StreamFrame sequence, estimated_total, early termination); §6.7 Aggregation queries (COUNT/SUM/AVG/MIN/MAX/COUNT_DISTINCT, group_by, having); ActionFrame §7.1 added `request_id`; SubscribeFrame §8.1 added `resume_from_seq`; §8.2 DiffFrame extension fields (monotonic seq, event_type, timestamp) and reconnection semantics; §9.1/9.2 added X-NWP-Request-ID; §9.4 HTTP mode error response format (application/nwp-error+json); §10 updated complete examples (including error response); §13.6 callback_url abuse prevention security section; 5 new error codes (AGGREGATE-UNSUPPORTED/-INVALID, STREAM-UNSUPPORTED, SUBSCRIBE-SEQ-TOO-OLD, task cancel series) |
| 0.3 | 2026-04-14 | SubscribeFrame (0x12); auto_anchor; Filter $not/$exists/$regex; ActionFrame callback_url/priority; system.task.*; NWM min_agent_version/rate_limits; §14.4/14.5 security sections |
| 0.2 | 2026-04-12 | Unified port 17433; AnchorFrame owned by Node; CGN metering; NPS status code mapping |
| 0.1 | 2026-04-10 | Initial specification |

---

*Attribution: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
