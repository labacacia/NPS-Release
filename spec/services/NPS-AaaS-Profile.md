English | [中文版](./NPS-AaaS-Profile.cn.md)

# NPS-AaaS Profile: Agent-as-a-Service Compliance Specification

**Status**: Draft
**Version**: 0.1
**Date**: 2026-04-15
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.4), NPS-2 (NWP v0.4), NPS-3 (NIP v0.2), NPS-5 (NOP v0.3)

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
| Incompatible APIs across Agent platforms | Unified Gateway Node entry + NWP standard frame protocol |
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
│  Gateway Node (new NWP type)    │  ← Service entry, external API
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

## 2. Gateway Node (NWP Extension)

### 2.1 Definition

Gateway Node is the fourth NWP node type, serving as the unified entry point for AaaS
services. It does not handle business logic directly but routes requests to the internal
NOP orchestration layer.

| Property | Value |
|----------|-------|
| Node type | `gateway` |
| NWM node_type | `"gateway"` |
| Frame entry | ActionFrame (0x11) |
| Internal conversion | ActionFrame → NOP TaskFrame (0x40) |

### 2.2 Gateway Node Responsibilities

| Responsibility | Protocol | Description |
|---------------|----------|-------------|
| **Authentication** | NIP | Verify consumer NID, check scope |
| **Service catalog** | NWP NWM | Expose available Actions via NWM manifest |
| **Request routing** | NOP | Convert ActionFrame to TaskFrame, decompose DAG |
| **Token metering** | NPT | Per-request Token Budget management |
| **Rate limiting** | NWP | NID-based rate limiting |
| **Observability** | NOP Context | Inject trace_id/span_id for full-chain tracing |

### 2.3 NWM Gateway Manifest Example

```json
{
  "nwm_version": "0.4",
  "node_type": "gateway",
  "node_id": "nwp://api.example.com/agent-service",
  "display_name": "Example AaaS Gateway",
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
    "min_nip_version": "0.2",
    "required_scopes": ["agent:invoke"]
  }
}
```

### 2.4 Request Flow: ActionFrame → TaskFrame

```
Consumer                    Gateway Node                    NOP Orchestrator
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

NWP v0.4 already supports the `vector_search` field (§6.4). The Vector Proxy Layer
only needs to implement this interface for seamless integration.

---

## 4. AaaS Compliance Requirements

### 4.1 Compliance Levels

| Level | Name | Requirements |
|-------|------|-------------|
| **Level 1** | Basic | Gateway Node + NIP auth + NWM service catalog |
| **Level 2** | Standard | Level 1 + NOP orchestration + OpenTelemetry tracing + Token Budget |
| **Level 3** | Advanced | Level 2 + Vector Proxy Layer + K-of-N fault tolerance + audit log |

### 4.2 Level 1 — Basic Compliance

| Req ID | Description | Protocol |
|--------|-------------|----------|
| L1-01 | MUST deploy Gateway Node as the sole service entry point | NWP |
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
Consumer Agent                Gateway Node              NOP              Vector Memory Node    Action Worker
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
| Gateway Node | NWP v0.4 | Yes: add `node_type: "gateway"` to NWM |
| Request routing | NOP v0.3 | No: reuses TaskFrame/DelegateFrame |
| Authentication | NIP v0.2 | No: reuses NID + scope |
| Vector queries | NWP v0.4 §6.4 | No: reuses vector_search |
| Vector Proxy | NWP + NCP | No: implementation-level middleware |
| Token metering | NPT v0.1 | No: reuses token_est |
| Audit trail | NOP v0.3 §8.3 | No: reuses context.trace_id |

**Only protocol change required**: Add `gateway` node type to NWP NWM (backward compatible).

---

## 7. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-04-15 | Initial draft: AaaS architecture overview, Gateway Node definition, Vector Proxy Layer design, three-tier compliance requirements |

---

*Attribution: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
