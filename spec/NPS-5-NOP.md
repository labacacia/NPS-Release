English | [中文版](./NPS-5-NOP.cn.md)

# NPS-5: Neural Orchestration Protocol (NOP)

**Spec Number**: NPS-5
**Status**: Proposed
**Version**: 0.7
**Date**: 2026-06-12
**Port**: 17433 (default, shared) / 17437 (optional dedicated)
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.9), NPS-2 (NWP v0.14), NPS-3 (NIP v0.10)
**Supersedes**: NCP AlignFrame (0x05)

> This document is the NOP detailed specification. For a suite overview see [NPS-0-Overview.md](NPS-0-Overview.md).

---

## 1. Terminology

Keywords "MUST", "MUST NOT", "SHOULD", "MAY" in this document are interpreted per RFC 2119.

---

## 2. Protocol Overview

NOP defines task dispatch, delegation, synchronization, and result aggregation for multi-Agent collaboration. It evolves NCP AlignFrame (0x05) into AlignStream (0x43), supporting Directed Acyclic Graph (DAG) task flows, cross-Agent intermediate result sharing, resource pre-flight checks, K-of-N sync barriers, and OpenTelemetry distributed tracing.

### 2.1 Roles

| Role | Description |
|------|-------------|
| **Orchestrator** | Initiates TaskFrames, decomposes and assigns subtasks, aggregates final results |
| **Worker Agent** | Executes subtasks, returns results via AlignStream |
| **Relay Agent** | Transparent forwarding without task execution (optional, for network zone isolation) |

### 2.2 Relationship to NCP AlignFrame

NCP AlignFrame (0x05) is **Deprecated**. AlignStream (0x43) adds:

- Task DAG context (task_id binding)
- Cross-Agent intermediate result sharing
- Timeout and retry semantics (per-node granularity)
- NIP identity binding (sender_nid verification at each delegation step)
- Token-level backpressure control

### 2.3 NOP Position in the NPS Stack

```
NOP (Multi-Agent Orchestration)
  ├── NWP (Data & Operation Access)
  │     └── NCP (Transport Frames & Encoding)
  └── NIP (Identity & Scope Validation)
```

The Orchestrator calls Worker Agent node operations via NWP ActionFrame; Worker Agents push intermediate/final results to the Orchestrator via AlignStream; every delegation step must pass NIP scope verification.

---

## 3. Frame Types

### 3.1 TaskFrame (0x40)

The complete task definition submitted by the Orchestrator to its runtime or a coordination node.

**Field Definitions**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x40` |
| `task_id` | string | Required | Task unique identifier (UUID v4) |
| `dag` | object | Required | DAG definition, see §3.1.1 |
| `timeout_ms` | uint32 | Optional | Overall task timeout (milliseconds), default 30000, max 3600000 (1 hour) |
| `max_retries` | uint8 | Optional | Global maximum retry count (failure of any single node beyond this causes overall failure), default 2 |
| `priority` | string | Optional | Task priority: `"low"` / `"normal"` (default) / `"high"` |
| `callback_url` | string | Optional | Callback URL on task completion/failure (`https://`, see §8.4) |
| `preflight` | bool | Optional | If true, perform a resource pre-flight check (§4) before execution, default false |
| `compensation_policy` | string | Optional | Saga compensation policy: `"best_effort"` (default) or `"strict"`, see §3.1.6 |
| `callback_secret` | string | Optional | HMAC-SHA256 key (base64url, 32 bytes) for webhook delivery signing (NOP v0.6). When present, the Orchestrator MUST compute `X-NPS-Signature: sha256=<HMAC-SHA256-hex>` over the raw JSON body and include it on every POST to `callback_url`. Receivers SHOULD reject callbacks missing this header with 400. Error code: `NOP-CALLBACK-HMAC-MISSING`. |
| `result_ttl_seconds` | uint32 | Optional | How long (seconds) the Orchestrator retains the task result after completion; default 3600. After TTL expires, result retrieval returns `NOP-TASK-RESULT-EXPIRED`. (NOP v0.7) |
| `context` | object | Optional | Pass-through context, see §3.1.2 |
| `request_id` | string | Optional | UUID v4 for request tracing |

#### §3.1.1 dag Field

A DAG consists of `nodes` (vertices) and `edges` (directed edges) describing subtask execution order and data flow.

**DAG Node Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Required | Node unique identifier (unique within the DAG) |
| `action` | string | Required | Operation URL (`nwp://...`) |
| `agent` | string | Required | NID of the Worker Agent to execute this node |
| `input_from` | array | Optional | List of upstream node IDs this node depends on; null or empty means a root node |
| `input_mapping` | object | Optional | Mapping of upstream output fields → this node's input params, see §3.1.3 |
| `timeout_ms` | uint32 | Optional | Per-node timeout (milliseconds); takes precedence over TaskFrame's global timeout_ms |
| `retry_policy` | object | Optional | Per-node retry policy, see §3.1.4 |
| `condition` | string | Optional | Condition expression (CEL subset); if false, this node is skipped (see §3.1.5) |
| `compensate_action` | string | Optional | NID URI of the compensating action invoked if this node must be reversed (see §3.1.6) |
| `compensate_params_mapping` | object | Optional | Maps this node's output field names to `compensate_action` input param names (see §3.1.6) |

**DAG Validation Rules**
- MUST be a Directed Acyclic Graph (no cycles; violation returns `NOP-TASK-DAG-CYCLE`)
- MUST have at least one root node (no incoming edges) and one terminal node (no outgoing edges)
- Maximum node count: **32** (violation returns `NOP-TASK-DAG-TOO-LARGE`)
- Maximum delegation chain depth: **3 levels** — each entity in the chain counts as one level (Orchestrator = level 1, Worker = level 2, Sub-Worker = level 3); a fourth entity (Sub-Sub-Worker) would exceed the limit. Equivalently: at most 2 `DelegateFrame` hops. Violation returns `NOP-DELEGATE-CHAIN-TOO-DEEP`.

**DAG Example**

```json
{
  "nodes": [
    {
      "id": "fetch",
      "action": "nwp://api.example.com/products/query",
      "agent":  "urn:nps:agent:...:fetcher",
      "timeout_ms": 5000
    },
    {
      "id": "analyze",
      "action": "nwp://ml.example.com/inference/invoke",
      "agent":  "urn:nps:agent:...:analyzer",
      "input_from": ["fetch"],
      "input_mapping": { "products": "$.fetch.data" },
      "retry_policy": { "max_retries": 3, "backoff": "exponential" }
    },
    {
      "id": "report",
      "action": "nwp://report.example.com/generate/invoke",
      "agent":  "urn:nps:agent:...:reporter",
      "input_from": ["analyze"],
      "input_mapping": { "analysis": "$.analyze.result" },
      "condition": "$.analyze.result.score > 0.7"
    }
  ],
  "edges": [
    { "from": "fetch",   "to": "analyze" },
    { "from": "analyze", "to": "report"  }
  ]
}
```

#### §3.1.2 context Field

| Field | Type | Description |
|-------|------|-------------|
| `session_id` | string | Agent session identifier (reused across requests) |
| `trace_id` | string | OpenTelemetry Trace ID (16-byte hex, 32 characters) |
| `span_id` | string | Current Span ID (8-byte hex, 16 characters) |
| `trace_flags` | uint8 | OpenTelemetry Trace Flags (e.g. `0x01` = sampled) |
| `baggage` | object | OpenTelemetry Baggage (key-value pairs, propagated to all subtasks) |
| `custom` | object | Application-defined context (passed through transparently; NOP does not interpret) |

Implementations SHOULD support OpenTelemetry W3C TraceContext format to enable visualization of multi-Agent task hop chains in existing monitoring systems (Jaeger / Zipkin / Tempo).

#### §3.1.3 input_mapping Field

`input_mapping` uses JSONPath expressions to map upstream node output fields to this node's input parameters:

```json
{
  "input_mapping": {
    "local_param_name": "$.upstream_node_id.result.field_path"
  }
}
```

- Key: parameter name in this node's ActionFrame.params
- Value: JSONPath expression; the root `$` represents the Orchestrator context (containing all upstream node results)
- When merging multiple upstream results, use an array: `["$.node_a.result", "$.node_b.result"]`

#### §3.1.4 retry_policy Field

| Field | Type | Description |
|-------|------|-------------|
| `max_retries` | uint8 | Maximum retries for this node (overrides TaskFrame's global max_retries) |
| `backoff` | string | Backoff strategy: `"fixed"` / `"linear"` / `"exponential"` (default) |
| `initial_delay_ms` | uint32 | First retry delay (milliseconds), default 1000 |
| `max_delay_ms` | uint32 | Maximum delay cap (milliseconds), default 30000 |
| `retry_on` | array | Error codes that trigger a retry; if omitted, retry on all failures |

Backoff formula: `delay = min(initial_delay_ms * backoff_factor^attempt, max_delay_ms)`
- `fixed`: factor=1; `linear`: factor=attempt; `exponential`: factor=2^attempt

#### §3.1.5 condition Field

`condition` uses a **CEL (Common Expression Language) subset**, supporting:

- Numeric comparisons: `>`, `>=`, `<`, `<=`, `==`, `!=`
- Boolean logic: `&&`, `||`, `!`
- JSONPath access to upstream results (using `$.<node_id>.<field>` syntax)
- String comparison and `in` operator

When `condition` evaluates to `false`, the node is skipped (status marked `SKIPPED`). If a terminal node is skipped, the TaskFrame ends with `COMPLETED` (not FAILED).

#### §3.1.6 Compensation Semantics (Saga)

DAG nodes that have produced externally observable side effects (charges, bookings, provisioning, message dispatch) MAY declare a **compensating action** that semantically reverses those effects. When a downstream failure forces the orchestrator to roll back, the compensating actions of already-completed predecessor nodes are invoked in reverse topological order — implementing the saga pattern.

**Per-Node Fields**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `compensate_action` | string | Optional | NID URI of the compensating action (`nwp://...`); if absent, the node is treated as non-reversible |
| `compensate_params_mapping` | object | Optional | Maps this node's output field names → compensate_action input param names. Keys are `compensate_action` parameter names; values are JSONPath expressions rooted at this node's result (e.g. `"$.charge_id"`) |

**TaskFrame-Level Field**

`compensation_policy` controls how compensation failures are handled:

| Value | Behavior |
|-------|----------|
| `"best_effort"` (default) | Compensation failures are logged; the orchestrator continues compensating remaining predecessors, then terminates the saga as FAILED |
| `"strict"` | Any compensation failure terminates the entire saga immediately with `NOP-COMPENSATION-FAILED`; remaining predecessors are NOT compensated |

**Saga Trigger**

If a node `N` reaches `FAILED` (after exhausting `retry_policy.max_retries`) and `N` has predecessors that successfully reached `COMPLETED`, the orchestrator SHOULD invoke each such predecessor's `compensate_action` in **reverse topological order** (most recent successful predecessor first). The orchestrator MUST:

1. Mark each compensating predecessor as `COMPENSATING` and dispatch a DelegateFrame with `action=compensate_action` and `params` derived from `compensate_params_mapping` applied to the predecessor's stored result.
2. On compensation success, transition the predecessor to `COMPENSATED`.
3. On compensation failure, transition the predecessor to `COMPENSATION_FAILED` and apply `compensation_policy` (above).
4. Skip predecessors that have no `compensate_action`. If `compensation_policy="strict"` AND any failed-node predecessor lacks a `compensate_action`, the orchestrator MUST return `NOP-COMPENSATION-NOT-SUPPORTED` (terminal) and not invoke any compensations for that saga.

Compensating actions MUST be idempotent — they MAY be retried per the predecessor's `retry_policy` and SHOULD use a stable `idempotency_key` derived from `<parent_task_id>:<node_id>:compensate`.

**Compensation Example**

```json
{
  "id": "charge",
  "action": "nwp://payments.example.com/charge/invoke",
  "agent": "urn:nps:agent:...:payments",
  "compensate_action": "nwp://payments.example.com/refund/invoke",
  "compensate_params_mapping": { "charge_id": "$.charge_id", "amount": "$.amount" }
}
```

If a downstream `ship` node fails after `charge` has completed, the orchestrator dispatches `nwp://payments.example.com/refund/invoke` with `{ "charge_id": <captured>, "amount": <captured> }`, transitioning the `charge` node `COMPLETED → COMPENSATING → COMPENSATED`.

**Complete TaskFrame Example**

```json
{
  "frame": "0x40",
  "task_id": "550e8400-e29b-41d4-a716-446655440000",
  "dag": {
    "nodes": [
      { "id": "fetch",   "action": "nwp://api.example.com/products/query",    "agent": "urn:nps:agent:...:fetcher",  "timeout_ms": 5000 },
      { "id": "analyze", "action": "nwp://ml.example.com/inference/invoke",   "agent": "urn:nps:agent:...:analyzer", "input_from": ["fetch"], "input_mapping": { "products": "$.fetch.data" } },
      { "id": "report",  "action": "nwp://report.example.com/generate/invoke","agent": "urn:nps:agent:...:reporter", "input_from": ["analyze"], "condition": "$.analyze.result.confidence > 0.8" }
    ],
    "edges": [
      { "from": "fetch",   "to": "analyze" },
      { "from": "analyze", "to": "report"  }
    ]
  },
  "timeout_ms": 60000,
  "max_retries": 2,
  "priority": "normal",
  "callback_url": "https://orchestrator.myapp.com/nop/callbacks",
  "preflight": true,
  "context": {
    "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
    "span_id":  "00f067aa0ba902b7",
    "trace_flags": 1,
    "session_id": "sess-abc123"
  },
  "request_id": "550e8400-e29b-41d4-a716-446655440001"
}
```

---

### 3.2 DelegateFrame (0x41)

The Orchestrator delegates a single DAG subtask to a Worker Agent.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x41` |
| `parent_task_id` | string | Required | Parent task_id |
| `subtask_id` | string | Required | Subtask unique identifier (UUID v4) |
| `node_id` | string | Required | Corresponding node id in the DAG |
| `target_agent_nid` | string | Required | NID of the Worker Agent being delegated to |
| `action` | string | Required | Operation URL (`nwp://`) |
| `params` | object | Optional | Operation parameters (processed via input_mapping) |
| `delegated_scope` | object | Required | Subset carved from the parent scope (MUST NOT be expanded) |
| `deadline_at` | string | Required | Subtask deadline (ISO 8601 UTC) |
| `idempotency_key` | string | Optional | Idempotency key (use the same value on retries) |
| `priority` | string | Optional | Inherited from TaskFrame.priority |
| `target_cluster_anchor` | string | Optional | NID of the cluster anchor to route this subtask to (cross-cluster delegation, NOP v0.6). When present, the Orchestrator MUST route the DelegateFrame to a Worker Agent registered under the specified cluster anchor. |
| `context` | object | Optional | Pass-through context (inherited from TaskFrame.context with span_id updated to the current Delegate span) |

**Scope Carving Principle**

The `delegated_scope`'s `nodes`, `actions`, and `max_token_budget` MUST all be subsets of the parent Agent's scope. The CA enforces this at signing time; violations are rejected with `NOP-DELEGATE-SCOPE-VIOLATION`.

**Worker Agent Rejection Response**

If a Worker Agent cannot accept the delegation (overloaded, insufficient capability, etc.), it responds with a CapsFrame:

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:delegate:rejected",
  "count": 1,
  "data": [{
    "subtask_id": "uuid-v4",
    "error": "NOP-DELEGATE-REJECTED",
    "reason": "capacity_exceeded",
    "retry_after_ms": 5000
  }]
}
```

---

### 3.3 SyncFrame (0x42)

A multi-Agent state synchronization barrier that waits for dependent subtasks to complete before proceeding. Supports K-of-N semantics.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x42` |
| `task_id` | string | Required | Parent task_id |
| `sync_id` | string | Required | Sync point unique identifier (UUID v4) |
| `wait_for` | array | Required | List of subtask_ids to wait for |
| `min_required` | uint32 | Optional | K-of-N: minimum number of subtasks that must succeed to proceed; omit or 0 means all must succeed (default) |
| `aggregate` | string | Optional | Result aggregation strategy: `"merge"` (default) / `"first"` / `"all"` / `"fastest_k"` / `"weighted_first_k"` / `"merge_all"`, see §3.3.1 and §3.3.2 |
| `timeout_ms` | uint32 | Optional | Wait timeout (milliseconds); returns `NOP-SYNC-TIMEOUT` on expiry |

#### §3.3.1 K-of-N Sync Semantics

The `min_required` field enables the following scenarios:

| Scenario | Configuration | Behavior |
|----------|--------------|----------|
| All must complete | `min_required` omitted or 0 | Wait for all `wait_for` subtasks to succeed |
| Any K complete | `min_required: K` | Continue immediately when K subtasks succeed; cancel the rest |
| Redundancy/fault tolerance | `min_required: N-1` (N-1 of N) | Proceed even if 1 node fails |

When K < N, once the barrier passes, the Orchestrator SHOULD send a cancel signal (DelegateFrame with `action="cancel"`) to any remaining incomplete subtasks.

#### §3.3.2 Aggregation Strategy Definitions

| Strategy | Description |
|----------|-------------|
| `merge` | Merge all successful subtask result fields into a single object (later keys overwrite earlier ones) |
| `first` | Use the result of the first subtask to complete successfully |
| `all` | Preserve all results as an array `[result_a, result_b, ...]` |
| `fastest_k` | Use the `min_required` fastest-completing results (merged in `all` format) |
| `weighted_first_k` | Select the `min_required` results with the highest confidence scores (requires `result.score` field in each subtask result); merge as `all` format (NOP v0.6) |
| `merge_all` | Merge all successful subtask results into a single object, preserving all fields; unlike `merge`, array-valued fields are concatenated rather than overwritten (NOP v0.6) |

**SyncFrame Completion Response (CapsFrame)**

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:sync:result",
  "count": 1,
  "data": [{
    "sync_id": "uuid-v4",
    "task_id": "uuid-v4",
    "status": "completed",
    "completed": ["subtask-a", "subtask-b"],
    "skipped":   ["subtask-c"],
    "failed":    [],
    "results": {
      "subtask-a": { ... },
      "subtask-b": { ... }
    },
    "aggregated": { ... }
  }]
}
```

---

### 3.4 AlignStream (0x43)

Directed task stream, replacing NCP AlignFrame (0x05). Carries DAG context and NIP identity binding.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `frame` | uint8 | Required | Fixed value `0x43` |
| `stream_id` | string | Required | Stream unique identifier (UUID v4) |
| `task_id` | string | Required | Associated parent task_id |
| `subtask_id` | string | Required | Associated subtask_id |
| `seq` | uint64 | Required | Message sequence number, strictly increasing from 0 |
| `payload_ref` | string | Optional | anchor_ref of the CapsFrame (intermediate result reference) |
| `data` | object | Optional | Intermediate result data |
| `window_size` | uint32 | Optional | Backpressure window size (unit: CGN Token count), see §3.4.1 |
| `is_final` | bool | Required | true indicates end of stream (final result frame) |
| `ack_seq` | uint64 | Optional | Acknowledge all frames with `seq ≤ ack_seq` received and processed. Used in the sliding-window protocol (NOP v0.6 §3.4.2). |
| `nak_seq` | uint64 | Optional | Negative-acknowledge: request retransmission from `nak_seq`. Sender MUST resend from `nak_seq`; if the frame is no longer available, sender returns `NOP-STREAM-NAK-UNRESOLVABLE`. |
| `sender_nid` | string | Required | Sender NID (receiver MUST verify it matches the connection identity) |
| `error` | object | Optional | Error information (may be present when is_final=true, indicating subtask failure) |

**error Object**

| Field | Type | Description |
|-------|------|-------------|
| `code` | string | NOP error code |
| `message` | string | Human-readable error description |
| `retryable` | bool | Whether the error is retryable |

#### §3.4.1 Token-Level Backpressure

`window_size` represents the maximum **CGN Token count** the receiver can currently process (not bytes), directly corresponding to the downstream inference endpoint's throughput capacity:

- Before sending each `data` frame, the sender SHOULD estimate the CGN cost of the frame and check the current window balance
- If the estimated cost exceeds the window balance, the sender SHOULD pause pushing and wait for the receiver to restore the window via a reverse AlignStream (only updating `window_size`)
- Window acknowledgement: after consuming, the receiver sends an AlignStream frame with `data=null, window_size=N` (where N is the newly available capacity) to restore the window

**Comparison with NCP StreamFrame**

| Dimension | StreamFrame (0x03) | AlignStream (0x43) |
|-----------|-------------------|--------------------|
| Use case | General data streams (NWP query results, etc.) | Multi-Agent task collaboration intermediate results |
| Context | None | Carries task_id + subtask_id |
| Identity binding | None | sender_nid mandatory verification |
| Backpressure unit | Bytes / frame count | CGN Token count |
| Error propagation | None | `error` field (task failure semantics) |

---

## 4. Resource Pre-flight

When TaskFrame has `preflight: true`, the Orchestrator sends lightweight probes to all Worker Agents before formally executing the DAG to confirm resource availability.

### 4.1 Pre-flight Flow

```
Orchestrator                        Worker Agent(s)
  │                                       │
  │── DelegateFrame(action="preflight") → │  Probe request (one per DAG node's Agent)
  │  ←── CapsFrame(preflight result) ─── │  Report availability
  │                                       │
  │  All Agents available → proceed       │
  │  Any Agent unavailable → abort        │
```

### 4.2 Pre-flight Request (DelegateFrame Extension)

When `action="preflight"`, `params` contains:

```json
{
  "estimated_npt": 1500,
  "required_capabilities": ["nwp:invoke", "ml:inference"],
  "action": "nwp://ml.example.com/inference/invoke"
}
```

### 4.3 Pre-flight Response (CapsFrame)

```json
{
  "frame": "0x04",
  "anchor_ref": "nps:system:preflight:result",
  "count": 1,
  "data": [{
    "agent_nid": "urn:nps:agent:...:analyzer",
    "available": true,
    "available_npt": 8000,
    "estimated_queue_ms": 200,
    "capabilities": ["nwp:invoke", "ml:inference"]
  }]
}
```

If `available: false`, the Orchestrator MUST abort the entire TaskFrame and return `NOP-RESOURCE-INSUFFICIENT`.

---

## 5. Task Execution State Machine

```
              ┌──────────┐
              │ PENDING  │  TaskFrame submitted, awaiting scheduling
              └────┬─────┘
                   │ Scheduling begins
                   ↓
           ┌──────────────┐
           │  PREFLIGHT   │  Resource pre-flight in progress (when preflight=true)
           └──────┬───────┘
                  │ Pre-flight passed
                  ↓
              ┌──────────┐
              │ RUNNING  │  Subtasks executing
              └────┬─────┘
                   │
         ┌─────────┼─────────┐
         ↓         ↓         ↓
  ┌────────────┐ ┌──────┐ ┌──────────┐
  │WAITING_SYNC│ │FAILED│ │CANCELLED │
  │(sync barrier)└──┬───┘ └──────────┘
  └──────┬─────┘   │ Exceeds max_retries
         │deps done │ Orchestrator notified of FAILED
         ↓         ↓
    ┌───────────┐
    │ COMPLETED │
    └───────────┘
```

Subtask states: `PENDING` → `RUNNING` → `COMPLETED` / `FAILED` / `CANCELLED` / `SKIPPED`

A `COMPLETED` subtask MAY additionally transition into the saga-compensation lifecycle when a downstream failure triggers rollback (see §3.1.6):

```
COMPLETED ──→ COMPENSATING ──→ COMPENSATED          (success — reversal acknowledged)
                  │
                  └────────→ COMPENSATION_FAILED    (terminal — reversal returned an error)
```

| State | Meaning |
|-------|---------|
| `COMPENSATING` | The node's `compensate_action` has been dispatched; awaiting result |
| `COMPENSATED` | The node was successfully reversed |
| `COMPENSATION_FAILED` | Terminal — the `compensate_action` returned an error and was not retried successfully |

`COMPENSATED` and `COMPENSATION_FAILED` are terminal saga states; they do NOT alter the parent TaskFrame's outcome (which remains `FAILED` once the original failure triggered the saga), but they MUST be reported in the final callback payload so callers can audit which side effects were reversed.

### 5.1 Retry Semantics

- Execute per the node's `retry_policy` (default: exponential backoff)
- Retries MUST use the same `subtask_id` and `idempotency_key`
- Once `max_retries` is exceeded, mark that subtask as FAILED
- Whether a subtask FAILED causes overall FAILED depends on SyncFrame `min_required`:
  - If failures > N - min_required, the overall task FAILED
  - Otherwise (K-of-N still satisfied), overall execution continues

### 5.2 Task Cancellation

At any time, the Orchestrator can cancel a task via:

1. **Direct disconnect**: Worker Agents MUST detect the connection closing and stop execution
2. **Send cancel DelegateFrame**: `action="cancel"`, `params: { "task_id": "...", "subtask_id": "..." }`
3. **Call NWP `system.task.cancel`** (if the node supports it)

Upon receiving a cancel signal, Worker Agents MUST stop execution and return `AlignStream(is_final=true, error.code="NOP-TASK-CANCELLED")`.

---

## 6. Complete Multi-Agent Collaboration Flow

```
Orchestrator                              Worker B (Data)    Worker C (Inference)
  │                                           │                   │
  │── TaskFrame(preflight=true) ─────────→   │                   │
  │      DelegateFrame(preflight) ─────────→ │                   │
  │      DelegateFrame(preflight) ────────────────────────────→  │
  │  ←── CapsFrame(available=true) ───────── │                   │
  │  ←── CapsFrame(available=true) ────────────────────────────  │
  │                                           │                   │
  │── DelegateFrame(fetch-data) ───────────→ │                   │
  │      Worker B calls NWP QueryFrame        │                   │
  │  ←── AlignStream(seq=0, data=products) ─ │                   │
  │  ←── AlignStream(is_final=true) ──────── │                   │
  │                                           │                   │
  │── DelegateFrame(analyze, products) ───────────────────────→  │
  │      Worker C calls inference node                            │
  │  ←── AlignStream(seq=0, progress=0.5) ─────────────────────  │
  │  ←── AlignStream(is_final=true, result) ───────────────────  │
  │                                           │                   │
  │── SyncFrame(wait_for=[fetch,analyze])     │                   │
  │   sync passed, aggregated result ready    │                   │
  │                                           │                   │
  │  → POST callback_url (task completion)    │                   │
```

---

## 7. Error Codes

| Error Code | NPS Status Code | Description |
|-----------|----------------|-------------|
| `NOP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | task_id does not exist |
| `NOP-TASK-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Overall task timeout |
| `NOP-TASK-DAG-INVALID` | `NPS-CLIENT-BAD-FRAME` | DAG format invalid (missing root/terminal node, field errors, etc.) |
| `NOP-TASK-DAG-CYCLE` | `NPS-CLIENT-BAD-FRAME` | DAG contains a cycle |
| `NOP-TASK-DAG-TOO-LARGE` | `NPS-CLIENT-BAD-FRAME` | DAG node count exceeds limit (default 32) |
| `NOP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | Task already completed; cannot resubmit |
| `NOP-TASK-CANCELLED` | `NPS-CLIENT-CONFLICT` | Task has been cancelled |
| `NOP-DELEGATE-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | delegated_scope exceeds parent Agent scope |
| `NOP-DELEGATE-REJECTED` | `NPS-CLIENT-UNPROCESSABLE` | Worker Agent rejected the delegation (insufficient capability or overloaded) |
| `NOP-DELEGATE-CHAIN-TOO-DEEP` | `NPS-CLIENT-BAD-PARAM` | Delegation chain depth exceeds the 3-level maximum (counted as entities: Orchestrator=1, Worker=2, Sub-Worker=3; a fourth entity triggers this error; at most 2 `DelegateFrame` hops permitted — see §3.1.1) |
| `NOP-DELEGATE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Subtask did not complete before deadline_at |
| `NOP-SYNC-TIMEOUT` | `NPS-SERVER-TIMEOUT` | SyncFrame wait for dependent tasks timed out |
| `NOP-SYNC-DEPENDENCY-FAILED` | `NPS-CLIENT-UNPROCESSABLE` | A dependent subtask failed (and failure count exceeds K-of-N tolerance) |
| `NOP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | AlignStream sequence number is non-contiguous |
| `NOP-STREAM-NID-MISMATCH` | `NPS-AUTH-UNAUTHENTICATED` | AlignStream sender_nid does not match connection identity |
| `NOP-RESOURCE-INSUFFICIENT` | `NPS-SERVER-UNAVAILABLE` | Pre-flight found one or more Worker Agents with insufficient resources (CGN / capability) |
| `NOP-CONDITION-EVAL-ERROR` | `NPS-CLIENT-BAD-PARAM` | DAG node condition expression evaluation failed (syntax error or reference to non-existent field) |
| `NOP-INPUT-MAPPING-ERROR` | `NPS-CLIENT-UNPROCESSABLE` | input_mapping JSONPath could not be resolved or target field does not exist |
| `NOP-COMPENSATION-FAILED` | `NPS-CLIENT-UNPROCESSABLE` | Terminal — a node's `compensate_action` returned an error during saga rollback (see §3.1.6) |
| `NOP-COMPENSATION-NOT-SUPPORTED` | `NPS-CLIENT-UNPROCESSABLE` | Terminal — a predecessor that must be compensated has no `compensate_action` and `compensation_policy="strict"` |
| `NOP-CALLBACK-HMAC-MISSING` | `NPS-AUTH-UNAUTHENTICATED` | Callback recipient rejected delivery because `X-NPS-Signature` header was absent; `callback_secret` was set but signature was not computed |
| `NOP-TASK-RESULT-EXPIRED` | `NPS-CLIENT-NOT-FOUND` | Task result requested after `result_ttl_seconds` elapsed; result no longer retained |
| `NOP-STREAM-NAK-UNRESOLVABLE` | `NPS-STREAM-SEQ-GAP` | NAK retransmission requested for a frame no longer available in sender's buffer (frame has been evicted) |
| `NOP-CLAIM-CONFLICT` | `NPS-CLIENT-CONFLICT` | TaskFrame already leased by a live runner lease (NPS-CR-0007 §4.2) |
| `NOP-SPAWN-SPEC-INVALID` | `NPS-CLIENT-BAD-PARAM` | `spawn_spec_ref` could not be resolved or failed SpawnSpec schema validation (NPS-CR-0007 §5) |
| `NOP-RUNTIME-IDLE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | L3 worker exceeded its idle timeout before completing the node (NPS-CR-0007 §6) |
| `NOP-RUNTIME-MAX-RUNTIME` | `NPS-SERVER-TIMEOUT` | L3 worker exceeded its max runtime before completing the node (NPS-CR-0007 §6) |

---

## 8. L3 Runtime Integration (nps-runner)

> Normative contract: **NPS-CR-0007** (`cr/NPS-CR-0007-nop-l3-runtime-integration.md`).
> This section is the in-spec summary; the CR carries the full field tables and conformance suite.

The NPS-Node Profile **L3 (on-demand / FaaS)** tier materialises an agent process only when a
task arrives. The runtime that does so is the `nps-runner` daemon. This section defines the
normative interface between `nps-runner` and the NOP orchestration layer.

### 8.1 Task-claim protocol

A runner leases the head of a per-NID inbox atomically: `{ task_id, runner_nid, lease_seconds
(clamped [10,600]), dedup_key = sha256(task_id ‖ dag_hash) }`. Outcomes:

- **Granted** — inbox marks the task `LEASED (runner_nid, lease_expiry)`; the runner MUST renew
  the lease before expiry while the node runs.
- **Conflict** — a live lease already exists ⇒ `NOP-CLAIM-CONFLICT`; the runner backs off.
- **Reclaim** — an expired lease is reclaimable; the `dedup_key` ensures a side-effect-bearing
  node already in a terminal state is **not** re-executed (at-least-once with dedup).

Result reporting is idempotent, keyed by `(task_id, node_id, dedup_key)`.

### 8.2 `spawn_spec_ref` content schema

`spawn_spec_ref` (NDP AnnounceFrame, NPS-4 §3.1) resolves to a **SpawnSpec**:
`{ image (OCI ref, required), command?, env?, resource_limits? {cpu, memory, cgn_budget},
idle_timeout_seconds?, max_runtime_seconds? }`. It MAY be an inline `spawnspec:` base64url-JSON
data URI or an `https://`/`nwp://` URL whose body MUST validate against the schema. Resolution
failure ⇒ `NOP-SPAWN-SPEC-INVALID`.

### 8.3 Lifecycle enforcement

| Limit | Precedence (high→low) | Terminal on breach |
|-------|-----------------------|--------------------|
| Idle timeout | SpawnSpec `idle_timeout_seconds` → runner policy | `NOP-RUNTIME-IDLE-TIMEOUT` |
| Max runtime | SpawnSpec `max_runtime_seconds` → runner policy | `NOP-RUNTIME-MAX-RUNTIME` |

On any terminal condition the runner PATCHes the node status into the orchestrator's task
store: `done → COMPLETED`, `failed`/idle/max-runtime → `FAILED`. A `FAILED` L3 node with a
`compensate_action` triggers the same reverse-topological saga rollback as L1/L2 (§3.1.6); L3
introduces no new saga path.

### 8.4 Conformance

L3 deployments are tested by `services/conformance/NPS-Node-L3.md` (`TC-N3-*`): claim conflict,
lease reclaim + dedup, spawn-spec resolution, idle/max-runtime breach, 3-node DAG end-to-end,
and L3 saga trigger.

---

## 9. Security Considerations

### 9.1 Task Injection Defense
The Orchestrator MUST verify that received TaskFrames originate from a trusted NID (via NIP certificate verification). Worker Agents SHOULD only accept DelegateFrames that have passed NIP verification and MUST reject delegations with non-matching or missing scope.

### 9.2 DAG Resource Limits
Implementations MUST enforce:
- Maximum DAG node count: **32**
- Maximum delegation chain depth: **3 levels**
- `condition` expression length: **≤ 512 characters**
- `input_mapping` JSONPath nesting depth: **≤ 8 levels**

### 9.3 Audit Trail
Every DelegateFrame execution SHOULD be written to an audit log containing:

- `sender_nid` (Orchestrator)
- `target_agent_nid` (Worker)
- `subtask_id`
- `parent_task_id`
- `timestamp`
- `trace_id` (from context, for correlation with OpenTelemetry systems)

### 9.4 callback_url Abuse Prevention
- TaskFrame `callback_url` MUST use the `https://` scheme
- The Orchestrator SHOULD perform SSRF checks on callback URLs (internal network addresses prohibited)
- On callback delivery failure, SHOULD apply exponential backoff retries (max 3 attempts), then abandon and log

### 9.5 Delegation Chain Security
Every delegation level must pass NIP CA verification that `delegated_scope` does not exceed the parent scope. Bypassing the CA to delegate with elevated permissions is not permitted.

---

## 10. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.7 | 2026-06-12 | **NPS-CR-0007 — NOP ↔ L3 runtime integration**: new §8 (task-claim protocol with lease + `dedup_key`; `spawn_spec_ref` SpawnSpec content schema; idle/max-runtime enforcement; idempotent result reporting); 4 new error codes (`NOP-CLAIM-CONFLICT`, `NOP-SPAWN-SPEC-INVALID`, `NOP-RUNTIME-IDLE-TIMEOUT`, `NOP-RUNTIME-MAX-RUNTIME`); new `services/conformance/NPS-Node-L3.md` (`TC-N3-*`); Security/Changelog renumbered §9/§10. Gates the `nps-runner` L3 FaaS runtime. |
| 0.7 | 2026-06-03 | `result_ttl_seconds` (uint32, default 3600) on TaskFrame — `NOP-TASK-RESULT-EXPIRED` after TTL; `NOP-STREAM-NAK-UNRESOLVABLE` error code for evicted-frame NAK retransmission |
| 0.6 | 2026-05-31 | `callback_secret` (HMAC-SHA256 key) on TaskFrame — `X-NPS-Signature` header on webhook callbacks, `NOP-CALLBACK-HMAC-MISSING`; `target_cluster_anchor` on DelegateFrame for cross-cluster routing; `weighted_first_k` / `merge_all` SyncFrame aggregate strategies (§3.3.2); `ack_seq` / `nak_seq` sliding-window ACK on AlignStream (§3.4.2) |
| 0.5 | 2026-05-10 | Compensation/saga semantics (issue #34): per-node `compensate_action` / `compensate_params_mapping`; TaskFrame-level `compensation_policy` (`best_effort` default / `strict`); subtask saga states `COMPENSATING` / `COMPENSATED` / `COMPENSATION_FAILED`; saga trigger in §3.1.6 (reverse-topological compensation of completed predecessors when a downstream node FAILED); 2 new error codes: `NOP-COMPENSATION-FAILED`, `NOP-COMPENSATION-NOT-SUPPORTED` |
| 0.4 | 2026-04-19 | Status / Depends-On version bumps; minor editorial alignment with NCP v0.7 / NWP v0.13 / NIP v0.9 |
| 0.3 | 2026-04-14 | DAG node granularity enhancements (per-node timeout/retry_policy/condition/input_mapping); §3.1.2 context supports OpenTelemetry W3C Trace (trace_id/span_id/trace_flags/baggage); §3.1.3 input_mapping JSONPath; §3.1.4 retry_policy (fixed/linear/exponential); §3.1.5 condition CEL subset; DelegateFrame adds idempotency_key/priority/context/node_id; SyncFrame adds min_required (K-of-N) and §3.3.1/§3.3.2 aggregation strategies; AlignStream adds subtask_id/error fields, §3.4.1 Token-level backpressure; §4 resource pre-flight protocol; §5 extended state machine (PREFLIGHT/SKIPPED) and task cancellation mechanism; §6 complete multi-Agent flow diagram; 5 new error codes (RESOURCE-INSUFFICIENT, CONDITION-EVAL-ERROR, INPUT-MAPPING-ERROR, DELEGATE-TIMEOUT, TASK-CANCELLED); §8.4 callback_url abuse prevention; Depends-On updated to NCP v0.7 / NWP v0.13 |
| 0.2 | 2026-04-12 | Unified port 17433; error codes use NPS status code mapping; completed error code list |
| 0.1 | 2026-04-10 | Initial spec: TaskFrame/DelegateFrame/SyncFrame/AlignStream, DAG execution model, supersedes NCP AlignFrame |

---

*Attribution: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
