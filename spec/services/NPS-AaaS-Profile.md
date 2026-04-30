English | [中文版](./NPS-AaaS-Profile.cn.md)

# NPS-AaaS Profile: Agent-as-a-Service Compliance Specification

**Status**: Proposed
**Version**: 0.4
**Date**: 2026-04-27
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.6), NPS-2 (NWP v0.8), NPS-3 (NIP v0.4), NPS-4 (NDP v0.5), NPS-5 (NOP v0.4)

> This specification defines compliance requirements for building Agent-as-a-Service (AaaS)
> platforms on the NPS protocol suite, covering three architectural layers: service entry,
> internal orchestration, and data access.

---

## 1. Overview

### 1.1 What is AaaS

Agent-as-a-Service is a cloud service model where providers expose AI Agent capabilities
through standardized APIs to consumers (other Agents or human applications), without
requiring consumers to understand internal implementation details.

### 1.2 What NPS-AaaS Solves

| Current Pain Point | NPS-AaaS Solution |
|-------------------|-------------------|
| Incompatible APIs across Agent platforms | Unified Anchor Node entry (cluster front door) + Bridge Node (NPS↔external) + NWP standard frame protocol |
| Non-standard, unobservable internal orchestration | NOP DAG orchestration + OpenTelemetry tracing |
| High token overhead for AI accessing traditional DBs | Vector Proxy Layer vectorization middleware |
| No Agent identity/permission standard | NIP NID identity + scope delegation chain |
| No service quality guarantees | NPT Token Budget + back-pressure control |

### 1.3 Architecture Overview

```
Consumer Agent
      │
      ▼
┌─────────────────────────────────┐
│  Anchor Node (NWP type)         │  ← Service entry, external API
│  • Authentication (NIP)         │
│  • Routing / Rate limit / NPT  │
│  • Service catalog (NWM)        │
└──────────┬──────────────────────┘
           │ DelegateFrame (NOP)
           ▼
┌─────────────────────────────────┐
│  Orchestration Layer (NOP)      │  ← Business coordination
│  • DAG task decomposition       │
│  • K-of-N sync / preflight      │
│  • Retry / timeout / cancel     │
└──────┬────────────┬─────────────┘
       │            │
       ▼            ▼
┌────────────┐ ┌────────────────────┐
│ Action Node│ │ Memory Node        │
│ (Worker)   │ │ + Vector Proxy     │  ← Traditional DB vectorized
└────────────┘ │   Layer            │
               └────────────────────┘
```

---

## 2. Anchor Node (NWP Extension)

> **Renamed from "Gateway Node" by [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md) in v1.0-alpha.3.** The role described in this section — stateless cluster entry point that routes inbound NWP `Action` frames into NOP `TaskFrame`s — is unchanged. The wire value `node_type: "gateway"` was removed and parsers MUST reject it. Implementations MAY add long-lived member-node registry behavior on top of the per-request stateless dispatch (NPS-2 §2.1 *Anchor Node — detailed semantics*); the topology read-back surface (`topology.snapshot` / `topology.stream`) is defined in [NPS-2 §12](../NPS-2-NWP.md) and is mandatory at L2 (see §4.3 below) per [NPS-CR-0002](../cr/NPS-CR-0002-anchor-topology-queries.md).

### 2.1 Definition

**Anchor Node** is an NWP node type serving as the unified entry point for AaaS
services. It does not handle business logic directly but routes requests to the internal
NOP orchestration layer. A single Anchor Node MAY simultaneously declare other roles
(e.g. `Memory Node`) via NDP `Announce.node_kind` array form (NPS-4 §3.1).

| Property | Value |
|----------|-------|
| Node type | `anchor` |
| NWM node_type | `"anchor"` |
| NDP node_kind | `"anchor"` (or array form including `"anchor"`) |
| Frame entry | ActionFrame (0x11) |
| Internal conversion | ActionFrame → NOP TaskFrame (0x40) |

### 2.2 Anchor Node Responsibilities

| Responsibility | Protocol | Description |
|---------------|----------|-------------|
| **Authentication** | NIP | Verify consumer NID, check scope |
| **Service catalog** | NWP NWM | Expose available Actions via NWM manifest |
| **Request routing** | NOP | Convert ActionFrame to TaskFrame, decompose DAG |
| **Token metering** | NPT | Per-request Token Budget management |
| **Rate limiting** | NWP | NID-based rate limiting |
| **Observability** | NOP Context | Inject trace_id/span_id for full-chain tracing |
| **Cluster registry** *(optional at L1, mandatory at L2)* | NDP + NWP | Track member nodes registered via `Announce.cluster_anchor`; surface them through `topology.snapshot` / `topology.stream` (NPS-2 §12) |

### 2.3 NWM Anchor Manifest Example

```json
{
  "nwm_version": "0.7",
  "node_type": "anchor",
  "node_id": "nwp://api.example.com/agent-service",
  "display_name": "Example AaaS Anchor",
  "capabilities": ["nop:orchestrate", "nwp:invoke", "nip:delegate"],
  "actions": [
    {
      "action_id": "analysis.run",
      "description": "Run a multi-step data analysis pipeline",
      "params_schema": { "$ref": "#/schemas/analysis_input" },
      "result_schema": { "$ref": "#/schemas/analysis_output" },
      "estimated_npt": 2000,
      "timeout_ms": 120000,
      "async": true
    }
  ],
  "rate_limits": {
    "requests_per_minute": 60,
    "max_concurrent": 10,
    "npt_per_hour": 100000
  },
  "auth": {
    "required": true,
    "min_nip_version": "0.4",
    "required_scopes": ["agent:invoke"]
  }
}
```

### 2.4 Request Flow: ActionFrame → TaskFrame

```
Consumer                    Anchor Node                    NOP Orchestrator
  │                              │                                │
  │── ActionFrame ──────────→   │                                │
  │   (action_id, params)       │                                │
  │                              │── verify NID + scope           │
  │                              │── lookup action → DAG template │
  │                              │── build TaskFrame ─────────→  │
  │                              │     (DAG, context, budget)     │
  │                              │                                │── DelegateFrame → Workers
  │                              │                                │── SyncFrame wait
  │                              │   ←── AlignStream(result) ──  │
  │  ←── CapsFrame(result) ──── │                                │
```

---

## 2A. Bridge Node (NWP Extension)

> Introduced by [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md) in v1.0-alpha.3 as the **second** half of the Gateway Node split. While Anchor Node sits at the front door of an NPS cluster (inbound NPS → routed NPS), Bridge Node sits at the *outbound* edge: NPS frames going **out** to non-NPS systems.

### 2A.1 Definition

**Bridge Node** is an NWP node type that translates NPS frames to and from non-NPS protocols. It is stateless per request and does not participate in cluster topology.

| Property | Value |
|----------|-------|
| Node type | `bridge` |
| NWM node_type | `"bridge"` |
| NDP node_kind | `"bridge"` (or array form including `"bridge"`) |
| NDP bridge_protocols | array of supported targets, e.g. `["http", "grpc"]` |
| Frame entry | ActionFrame (0x11) carrying `bridge_target` |

> **Direction note.** Bridge Node carries the **NPS → external** direction. The legacy `compat/*-bridge` packages historically published under LabAcacia (MCP / A2A / gRPC) carry the **inverse** direction (external → NPS) and have been renamed `compat/*-ingress` to remove the naming collision. See [NPS-CR-0001 §3.2](../cr/NPS-CR-0001-anchor-bridge-split.md) and CR-0001 §6.

### 2A.2 Bridge Node Responsibilities

| Responsibility | Protocol | Description |
|---------------|----------|-------------|
| **NPS frame ingest** | NWP | Accept ActionFrame containing `bridge_target` (target external endpoint + protocol selector) |
| **Outbound translation** | HTTP / gRPC / MCP / A2A | Encode the NPS payload as the target protocol's request format |
| **Response translation** | NWP | Decode the target's response and return as `CapsFrame` |
| **Authentication relay** *(optional)* | NIP / vendor | Forward NIP credentials as the target protocol's auth token (where mappable), or use vendor-side credentials configured per Bridge instance |
| **Observability** | NOP Context | Annotate spans with the target protocol + endpoint so end-to-end traces traverse the boundary cleanly |

### 2A.3 Standard Targets at v1.0-alpha.3

The set below is the minimum that reference Bridge Node implementations are expected to support; concrete `bridge_target` schemas per target protocol are deferred to follow-up CRs.

| Target | NDP `bridge_protocols` value | Notes |
|--------|------------------------------|-------|
| HTTP / HTTPS | `"http"` | REST and streaming |
| gRPC | `"grpc"` | Unary and streaming |
| Model Context Protocol | `"mcp"` | LLM-tool integration target |
| Agent-to-Agent | `"a2a"` | Google A2A v0.2 |

Additional protocols MAY be registered through future CRs by adding new `bridge_protocols` values.

### 2A.4 Removed: Gateway Node

> **Gateway Node** (removed in v1.0-alpha.3) — the role formerly known as "Gateway Node" lives on as **Anchor Node** (this section's §2). The new **Bridge Node** name was introduced because the term "bridge" had already been informally taken by `compat/*-bridge`; those packages have been renamed `compat/*-ingress`. See NPS-CR-0001 for full rationale.

---

## 3. Vector Proxy Layer (Vectorization Middleware)

### 3.1 Core Concept

Add a vector proxy layer in front of traditional databases (RDS/NoSQL):

```
AI Agent ←→ Vector Proxy Layer ←→ Traditional Database
           (vector space)          (SQL/document)
```

- **Write path**: Structured data → generate embedding vectors → store in vector index + write raw data to DB
- **Query path**: Agent semantic query → vector similarity search → return compressed vector representation + key fields
- **Effect**: AI no longer needs to parse full SQL result sets; consumes vectorized summaries directly, dramatically reducing token usage

### 3.2 Architecture

```
                    NWP Memory Node Interface
                          │
               ┌──────────┴──────────┐
               │  Vector Proxy Layer  │
               │                      │
               │  ┌────────────────┐  │
               │  │ Embedding Engine│  │  ← Embedding model (local/remote)
               │  └───────┬────────┘  │
               │          │           │
               │  ┌───────┴────────┐  │
               │  │ Vector Index   │  │  ← ANN index (HNSW/IVF)
               │  │ (memory/disk)  │  │
               │  └───────┬────────┘  │
               │          │           │
               │  ┌───────┴────────┐  │
               │  │ Schema Mapper  │  │  ← DB schema ↔ AnchorFrame mapping
               │  └────────────────┘  │
               └──────────┬───────────┘
                          │
               ┌──────────┴──────────┐
               │   Traditional DB    │
               │  (PostgreSQL/MySQL/  │
               │   MongoDB/...)       │
               └─────────────────────┘
```

### 3.3 Operating Modes

#### 3.3.1 Schema Mapping (Startup)

1. Scan target database schema (tables, columns, types, relationships)
2. Auto-generate NWP AnchorFrame schema
3. Generate embedding vectors for text columns, build ANN index
4. Register as NWP Memory Node (with `vector_search` capability)

#### 3.3.2 Query (Runtime)

| Query Mode | Use Case | Token Savings |
|-----------|----------|--------------|
| **Vector semantic query** | "Find records similar to X" | ~70-80% (return only top-K summary vectors) |
| **Structured query + vector summary** | Precise filter + AI-readable summary | ~40-60% (filter then vectorize-compress) |
| **Passthrough query** | Need complete raw data | 0% (fallback to traditional mode) |

#### 3.3.3 Response Format

```json
{
  "frame": "0x04",
  "anchor_ref": "sha256:...",
  "count": 5,
  "data": [
    {
      "_id": "product:1001",
      "_score": 0.95,
      "_embedding": [0.12, -0.34, ...],
      "name": "Widget Pro",
      "price": 29.99
    }
  ],
  "vector_meta": {
    "model": "text-embedding-3-small",
    "dimensions": 256,
    "index_type": "hnsw"
  },
  "token_est": 85
}
```

Compared to returning full row data with dozens of irrelevant columns, vector mode
returns only top-K similar results + embedding vectors + key fields, reducing token_est
from hundreds to double digits.

### 3.4 Integration with NWP QueryFrame

The Vector Proxy Layer is fully transparent to consumers. Standard NWP QueryFrame:

```json
{
  "frame": "0x10",
  "anchor_ref": "sha256:...",
  "vector_search": {
    "field": "_embedding",
    "vector": [0.11, -0.33, ...],
    "top_k": 5,
    "metric": "cosine"
  },
  "fields": ["name", "price", "_score"]
}
```

NWP v0.5 already supports the `vector_search` field (§6.4). The Vector Proxy Layer
only needs to implement this interface for seamless integration.

---

## 4. AaaS Compliance Requirements

### 4.1 Compliance Levels

| Level | Name | Requirements |
|-------|------|-------------|
| **Level 1** | Basic | Anchor Node + NIP auth + NWM service catalog |
| **Level 2** | Standard | Level 1 + NOP orchestration + OpenTelemetry tracing + Token Budget |
| **Level 3** | Advanced | Level 2 + Vector Proxy Layer + K-of-N fault tolerance + audit log |

### 4.2 Level 1 — Basic Compliance

| Req ID | Description | Protocol |
|--------|-------------|----------|
| L1-01 | MUST deploy Anchor Node as the sole service entry point | NWP |
| L1-02 | MUST verify consumer NID via NIP | NIP |
| L1-03 | MUST publish NWM manifest with all available Actions | NWP |
| L1-04 | MUST support NPS unified port 17433 | NCP |
| L1-05 | MUST return NPS standard status codes and error frames | NCP |
| L1-06 | SHOULD support both HTTP mode and native mode dual transport | NCP |

### 4.3 Level 2 — Standard Compliance

| Req ID | Description | Protocol |
|--------|-------------|----------|
| L2-01 | MUST use NOP TaskFrame for internal task orchestration | NOP |
| L2-02 | MUST inject OpenTelemetry trace in TaskFrame.context | NOP |
| L2-03 | MUST support NPT Token Budget with token_est in responses | NPT |
| L2-04 | MUST support NOP preflight mechanism | NOP |
| L2-05 | MUST implement NOP retry and timeout semantics | NOP |
| L2-06 | SHOULD support async Actions (ActionFrame.async=true) | NWP |
| L2-07 | SHOULD implement AlignStream back-pressure control | NOP |
| L2-08 | MUST implement `topology.snapshot` and `topology.stream` reserved query types on Anchor Nodes that maintain a member registry, per [NPS-2 §12](../NPS-2-NWP.md). The version counter MUST be monotonic per Anchor lifetime and SHOULD survive in-process restarts via rebase + `anchor_state.version_rebased` event. | NWP §12 |

### 4.4 Level 3 — Advanced Compliance

| Req ID | Description | Protocol |
|--------|-------------|----------|
| L3-01 | MUST deploy Vector Proxy Layer for vectorized queries | NWP |
| L3-02 | MUST support NWP vector_search interface | NWP §6.4 |
| L3-03 | MUST implement K-of-N sync fault tolerance (SyncFrame.min_required) | NOP |
| L3-04 | MUST maintain audit logs (NOP §8.3) | NOP |
| L3-05 | MUST implement scope delegation chain security (max 3 levels) | NIP + NOP |
| L3-06 | SHOULD support automatic schema discovery (DB schema → AnchorFrame) | NWP |
| L3-07 | SHOULD support hot vector index updates (incremental rebuild on data changes) | Vector Proxy |

---

## 5. End-to-End Scenario

### Scenario: Consumer Agent Requests Data Analysis Service

```
Consumer Agent                Anchor Node               NOP              Vector Memory Node    Action Worker
     │                             │                     │                      │                   │
     │── ActionFrame ────────→    │                     │                      │                   │
     │   action: analysis.run     │                     │                      │                   │
     │   params: {query: "..."}   │                     │                      │                   │
     │                             │── verify NID ──→   │                      │                   │
     │                             │── build DAG ──→    │                      │                   │
     │                             │   TaskFrame         │                      │                   │
     │                             │   ┌─────────┐      │                      │                   │
     │                             │   │ fetch    │──→   │── DelegateFrame ──→ │                   │
     │                             │   │ analyze  │      │                      │── QueryFrame ──→ DB
     │                             │   │ summarize│      │                      │  (vector_search)  │
     │                             │   └─────────┘      │  ←── CapsFrame ───── │  (top-K, 85 NPT) │
     │                             │                     │                      │                   │
     │                             │                     │── DelegateFrame ─────────────────────→  │
     │                             │                     │                                          │
     │                             │                     │  ←── AlignStream(result) ──────────────  │
     │                             │                     │                      │                   │
     │  ←── CapsFrame ──────────  │  ←── result ────── │                      │                   │
     │   (vectorized summary,      │                     │                      │                   │
     │    85+50 NPT)               │                     │                      │                   │
```

Traditional approach: fetch returns full row data ~500 NPT → analyze processes ~300 NPT = **800+ NPT**
AaaS vectorized approach: fetch returns top-K vector summary ~85 NPT → analyze ~50 NPT = **~135 NPT**

**Token savings ~83%**.

---

## 6. Relationship with Existing NPS Protocols

| Profile Component | NPS Protocol Dependency | Protocol Change Required |
|------------------|------------------------|------------------------|
| Anchor Node | NWP v0.8 | Yes: rename `node_type: "gateway"` to `"anchor"` (CR-0001); legacy `"gateway"` rejected. L2 cluster registry surfaces via `topology.snapshot` / `topology.stream` (CR-0002) |
| Bridge Node | NWP v0.8 | Yes: new `node_type: "bridge"` value (CR-0001) |
| Request routing | NOP v0.4 | No: reuses TaskFrame/DelegateFrame |
| Authentication | NIP v0.3 | No: reuses NID + scope |
| Vector queries | NWP v0.5 §6.4 | No: reuses vector_search |
| Vector Proxy | NWP + NCP | No: implementation-level middleware |
| Token metering | NPT v0.1 | No: reuses token_est |
| Audit trail | NOP v0.4 §8.3 | No: reuses context.trace_id |

**Protocol changes required**: NWP NWM `node_type` enum must accept `"anchor"` and `"bridge"` (replacing the removed `"gateway"`); NDP AnnounceFrame gains `node_kind`/`cluster_anchor`/`bridge_protocols` fields. All additive at the wire layer except the `"gateway"` removal — see [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md).

---

## 7. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.4 | 2026-04-27 | New L2-08 requirement (§4.3) mandating `topology.snapshot` / `topology.stream` on Anchor Nodes that maintain a member registry, per [NPS-CR-0002](../cr/NPS-CR-0002-anchor-topology-queries.md) and [NPS-2 §12](../NPS-2-NWP.md). §2 intro placeholder ("reserved for v1.0-alpha.4") replaced with concrete L2 mandate. §2.2 cluster-registry row promoted from "optional" to "optional at L1, mandatory at L2". §6 protocol-relationship table re-pointed at NWP v0.8 to pick up the new §12 surface. Depends-On bumped: NPS-2 (NWP v0.7 → v0.8). |
| 0.3 | 2026-04-26 | **Breaking** (per [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md)). §2 renamed Gateway Node → **Anchor Node**, all wire/manifest examples updated (`node_type: "anchor"`, `nwm_version: "0.7"`, `min_nip_version: "0.4"`); new §2.2 row for optional cluster registry. New §2A **Bridge Node** (NPS↔non-NPS protocol translation, `bridge_protocols` declaration, direction-vs-`compat/*-bridge` clarification). §1.2 / §1.3 architecture diagram and §6 protocol-relationship table updated; L1 §4.2 / §5 references re-pointed to Anchor Node. §6 protocol-change line replaced "add gateway" with "rename gateway → anchor + new bridge". Depends-On upgraded to NCP v0.6 (RFC-0001) + NWP v0.7 (CR-0001) + NIP v0.4 (RFC-0003) + NDP v0.5 (CR-0001 fields). |
| 0.1 | 2026-04-15 | Initial draft: AaaS architecture overview, Gateway Node definition, Vector Proxy Layer design, three-tier compliance requirements |

---

*Attribution: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
