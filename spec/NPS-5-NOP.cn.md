[English Version](./NPS-5-NOP.md) | 中文版

# NPS-5: Neural Orchestration Protocol (NOP)

**Spec Number**: NPS-5  
**Status**: Proposed  
**Version**: 0.4  
**Date**: 2026-04-19  
**Port**: 17433（默认，共用）/ 17437（可选独立）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.5), NPS-2 (NWP v0.5), NPS-3 (NIP v0.3)  
**Supersedes**: NCP AlignFrame (0x05)  

> 本文档为 NOP 详细规范。套件总览见 [NPS-0-Overview.cn.md](NPS-0-Overview.cn.md)。

---

## 1. Terminology

本文档中的关键字 "MUST"、"MUST NOT"、"SHOULD"、"MAY" 按照 RFC 2119 解释。

---

## 2. 协议概述

NOP 定义多 Agent 协作的任务分发、委托、同步与结果汇聚。基于 NCP AlignFrame (0x05) 演进为 AlignStream (0x43)，支持有向无环图（DAG）式任务流、跨 Agent 中间结果共享、资源预检、K-of-N 同步屏障，以及 OpenTelemetry 分布式追踪。

### 2.1 角色

| 角色 | 描述 |
|------|------|
| **Orchestrator** | 发起 TaskFrame，分解并分配子任务，汇聚最终结果 |
| **Worker Agent** | 执行子任务，通过 AlignStream 返回中间或最终结果 |
| **Relay Agent** | 透明转发，不执行任务（可选，用于隔离网络区域）|

### 2.2 与 NCP AlignFrame 的关系

NCP AlignFrame (0x05) 已标记为 **Deprecated**。AlignStream (0x43) 在其基础上增加：

- 任务 DAG 上下文（task_id 绑定）
- 跨 Agent 中间结果共享
- 超时与重试语义（per-node 粒度）
- NIP 身份绑定（每个委托步骤验证 sender_nid）
- Token 级背压控制

### 2.3 NOP 在 NPS 协议栈中的位置

```
NOP（多 Agent 编排）
  ├── NWP（数据与操作访问）
  │     └── NCP（传输帧 & 编码）
  └── NIP（身份认证 & scope 校验）
```

Orchestrator 通过 NWP ActionFrame 调用 Worker Agent 的节点操作；Worker Agent 通过 AlignStream 向 Orchestrator 推送中间/最终结果；每一步委托均须通过 NIP 验证 scope。

---

## 3. 帧类型

### 3.1 TaskFrame (0x40)

Orchestrator 向自身运行时或协调节点提交的整体任务定义。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x40` |
| `task_id` | string | 必填 | 任务唯一标识（UUID v4）|
| `dag` | object | 必填 | DAG 定义，见 §3.1.1 |
| `timeout_ms` | uint32 | 可选 | 整体任务超时（毫秒），默认 30000，最大 3600000（1 小时）|
| `max_retries` | uint8 | 可选 | 全局最大重试次数（单个节点超过时整体失败），默认 2 |
| `priority` | string | 可选 | 任务优先级：`"low"` / `"normal"`（默认）/ `"high"` |
| `callback_url` | string | 可选 | 任务完成/失败时的回调 URL（`https://`，见 §7.4）|
| `preflight` | bool | 可选 | true 时先执行资源预检（§5），通过后再正式执行，默认 false |
| `context` | object | 可选 | 透传上下文，见 §3.1.2 |
| `request_id` | string | 可选 | UUID v4，用于请求追踪 |

#### §3.1.1 dag 字段

DAG 由 `nodes`（顶点）和 `edges`（有向边）组成，描述子任务的执行顺序与数据流向。

**DAG 节点字段**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `id` | string | 必填 | 节点唯一标识（DAG 内唯一）|
| `action` | string | 必填 | 操作 URL（`nwp://...`）|
| `agent` | string | 必填 | 执行此节点的 Worker Agent NID |
| `input_from` | array | 可选 | 依赖的上游节点 id 列表；空或 null 表示起点节点 |
| `input_mapping` | object | 可选 | 上游输出字段 → 本节点输入参数的映射，见 §3.1.3 |
| `timeout_ms` | uint32 | 可选 | 本节点超时（毫秒）；优先级高于 TaskFrame 全局 timeout_ms |
| `retry_policy` | object | 可选 | 本节点重试策略，见 §3.1.4 |
| `condition` | string | 可选 | 条件表达式（CEL 子集），为 false 时跳过本节点（见 §3.1.5）|

**DAG 校验规则**
- MUST 是有向无环图（不含环路，违反返回 `NOP-TASK-DAG-CYCLE`）
- MUST 至少有一个起点节点（无入边）和一个终点节点（无出边）
- 最大节点数：**32**（违反返回 `NOP-TASK-DAG-TOO-LARGE`）
- 最大委托链深度：**3 层**（Orchestrator → Worker → Sub-Worker，违反返回 `NOP-DELEGATE-CHAIN-TOO-DEEP`）

**DAG 示例**

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

#### §3.1.2 context 字段

| 字段 | 类型 | 描述 |
|------|------|------|
| `session_id` | string | Agent 会话标识（跨请求复用）|
| `trace_id` | string | OpenTelemetry Trace ID（16 字节十六进制，32 字符）|
| `span_id` | string | 当前 Span ID（8 字节十六进制，16 字符）|
| `trace_flags` | uint8 | OpenTelemetry Trace Flags（如 `0x01` = sampled）|
| `baggage` | object | OpenTelemetry Baggage（键值对，透传至所有子任务）|
| `custom` | object | 应用自定义上下文（透明传递，NOP 不解析）|

实现 SHOULD 支持 OpenTelemetry W3C TraceContext 格式，以便在现有监控系统（Jaeger / Zipkin / Tempo）中可视化多 Agent 任务跳转链路。

#### §3.1.3 input_mapping 字段

`input_mapping` 使用 JSONPath 表达式将上游节点的输出结果字段映射到本节点的输入参数：

```json
{
  "input_mapping": {
    "local_param_name": "$.upstream_node_id.result.field_path"
  }
}
```

- 键：本节点 ActionFrame.params 中的参数名
- 值：JSONPath 表达式，根节点 `$` 表示 Orchestrator 上下文（包含所有上游节点的结果）
- 多个上游结果需合并时，可使用 `["$.node_a.result", "$.node_b.result"]` 数组

#### §3.1.4 retry_policy 字段

| 字段 | 类型 | 描述 |
|------|------|------|
| `max_retries` | uint8 | 最大重试次数（覆盖 TaskFrame 全局 max_retries）|
| `backoff` | string | 退避策略：`"fixed"` / `"linear"` / `"exponential"`（默认）|
| `initial_delay_ms` | uint32 | 首次重试延迟（毫秒），默认 1000 |
| `max_delay_ms` | uint32 | 最大延迟上限（毫秒），默认 30000 |
| `retry_on` | array | 触发重试的错误码列表；省略时对所有失败重试 |

退避公式：`delay = min(initial_delay_ms * backoff_factor^attempt, max_delay_ms)`
- `fixed`: factor=1；`linear`: factor=attempt；`exponential`: factor=2^attempt

#### §3.1.5 condition 字段

`condition` 使用 **CEL（Common Expression Language）子集**，支持：

- 数值比较：`>`, `>=`, `<`, `<=`, `==`, `!=`
- 布尔逻辑：`&&`, `||`, `!`
- JSONPath 访问上游结果（使用 `$.<node_id>.<field>` 语法）
- 字符串比较和 `in` 操作

condition 为 `false` 时，节点被跳过（状态标记为 `SKIPPED`）。若终点节点被跳过，TaskFrame 以 `COMPLETED` 结束（非 FAILED）。

**完整 TaskFrame 示例**

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

Orchestrator 将 DAG 中的单个子任务委托给 Worker Agent 执行。

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x41` |
| `parent_task_id` | string | 必填 | 父任务 task_id |
| `subtask_id` | string | 必填 | 子任务唯一标识（UUID v4）|
| `node_id` | string | 必填 | 对应 DAG 中的节点 id |
| `target_agent_nid` | string | 必填 | 被委托 Worker Agent 的 NID |
| `action` | string | 必填 | 操作 URL（`nwp://`）|
| `params` | object | 可选 | 操作参数（经 input_mapping 处理后的结果）|
| `delegated_scope` | object | 必填 | 从父 scope 裁剪的子集（MUST NOT 扩大）|
| `deadline_at` | string | 必填 | 子任务截止时间（ISO 8601 UTC）|
| `idempotency_key` | string | 可选 | 幂等键（重试时使用相同的值）|
| `priority` | string | 可选 | 继承自 TaskFrame.priority |
| `context` | object | 可选 | 透传上下文（从 TaskFrame.context 继承，span_id 更新为当前 Delegate span）|

**Scope 裁剪原则**

`delegated_scope` 中的 `nodes`、`actions`、`max_token_budget` 均 MUST 是父 Agent scope 的子集。CA 在签发时强制校验，违反则拒绝并返回 `NOP-DELEGATE-SCOPE-VIOLATION`。

**Worker Agent 拒绝响应**

若 Worker Agent 无法接受委托（负载、能力不足等），响应 CapsFrame：

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

多 Agent 状态同步屏障，等待依赖子任务完成后再继续，支持 K-of-N 语义。

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x42` |
| `task_id` | string | 必填 | 父任务 task_id |
| `sync_id` | string | 必填 | 同步点唯一标识（UUID v4）|
| `wait_for` | array | 必填 | 需等待的 subtask_id 列表 |
| `min_required` | uint32 | 可选 | K-of-N：最少需要多少个子任务成功才继续；省略或 0 表示全部成功（默认）|
| `aggregate` | string | 可选 | 结果聚合策略：`"merge"`（默认）/ `"first"` / `"all"` / `"fastest_k"`，见 §3.3.1 |
| `timeout_ms` | uint32 | 可选 | 等待超时（毫秒），超时返回 `NOP-SYNC-TIMEOUT` |

#### §3.3.1 K-of-N 同步语义

`min_required` 字段支持以下场景：

| 场景 | 配置 | 行为 |
|------|------|------|
| 全部完成 | `min_required` 省略或 0 | 等待所有 wait_for 子任务成功 |
| 任意 K 个完成 | `min_required: K` | K 个子任务成功后立即继续，其余取消 |
| 冗余容错 | `min_required: N-1`（N-1 of N）| 允许 1 个节点失败仍继续 |

K < N 时，同步屏障通过后，Orchestrator SHOULD 向剩余未完成子任务发送取消信号（DelegateFrame with action="cancel"）。

#### §3.3.2 聚合策略定义

| 策略 | 描述 |
|------|------|
| `merge` | 将所有成功子任务的结果字段合并为单个对象（相同键后者覆盖前者）|
| `first` | 取第一个成功完成的子任务结果 |
| `all` | 将所有结果保留为数组 `[result_a, result_b, ...]` |
| `fastest_k` | 取最先完成的 `min_required` 个结果（合并为 `all` 格式）|

**SyncFrame 完成响应（CapsFrame）**

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

有向任务流，替代 NCP AlignFrame (0x05)。携带 DAG 上下文和 NIP 身份绑定。

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x43` |
| `stream_id` | string | 必填 | 流唯一标识（UUID v4）|
| `task_id` | string | 必填 | 关联的父任务 task_id |
| `subtask_id` | string | 必填 | 关联的子任务 subtask_id |
| `seq` | uint64 | 必填 | 消息序号，严格递增，从 0 开始 |
| `payload_ref` | string | 可选 | CapsFrame 的 anchor_ref（中间结果引用）|
| `data` | object | 可选 | 中间结果数据 |
| `window_size` | uint32 | 可选 | 背压窗口大小（单位：NPT Token 数），见 §3.4.1 |
| `is_final` | bool | 必填 | true 表示流结束（最终结果帧）|
| `sender_nid` | string | 必填 | 发送方 NID（接收方 MUST 验证与连接身份一致）|
| `error` | object | 可选 | 错误信息（is_final=true 时可携带，表示子任务失败）|

**error 对象**

| 字段 | 类型 | 描述 |
|------|------|------|
| `code` | string | NOP 错误码 |
| `message` | string | 人类可读错误描述 |
| `retryable` | bool | 是否可重试 |

#### §3.4.1 Token 级背压控制

`window_size` 表示接收方当前可以处理的 **NPT Token 数量上限**（而非字节数），直接对应下游推理端的处理能力：

- 发送方在发送每帧 `data` 前，SHOULD 估算本帧 NPT 消耗，并检查当前窗口余量
- 若估算消耗超出余量，SHOULD 暂停推送，等待接收方通过反向 AlignStream（仅更新 window_size）恢复窗口
- 窗口确认机制：接收方消费后，发送 `data=null, window_size=N`（N 为新增可用量）的 AlignStream 帧恢复窗口

**与 NCP StreamFrame 的区别**

| 维度 | StreamFrame (0x03) | AlignStream (0x43) |
|------|-------------------|--------------------|
| 场景 | 通用数据流（NWP 查询结果等）| 多 Agent 任务协作中间结果 |
| 上下文 | 无 | 携带 task_id + subtask_id |
| 身份绑定 | 无 | sender_nid 强制验证 |
| 背压单位 | 字节 / 帧数 | NPT Token 数 |
| 错误传播 | 无 | error 字段（任务失败语义）|

---

## 4. 资源预检（Pre-flight）

当 TaskFrame `preflight: true` 时，Orchestrator 在正式执行 DAG 前，向各 Worker Agent 发送轻量级探针，确认资源可用性。

### 4.1 预检流程

```
Orchestrator                        Worker Agent(s)
  │                                       │
  │── DelegateFrame(action="preflight") → │  探针请求（每个 DAG 节点对应的 Agent）
  │  ←── CapsFrame(preflight result) ─── │  回报可用性
  │                                       │
  │  所有 Agent 可用 → 正式执行           │
  │  任意 Agent 不可用 → 中止并返回错误   │
```

### 4.2 预检请求（DelegateFrame 扩展）

`action="preflight"` 时，`params` 包含：

```json
{
  "estimated_npt": 1500,
  "required_capabilities": ["nwp:invoke", "ml:inference"],
  "action": "nwp://ml.example.com/inference/invoke"
}
```

### 4.3 预检响应（CapsFrame）

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

若 `available: false`，Orchestrator MUST 中止整个 TaskFrame 并返回 `NOP-RESOURCE-INSUFFICIENT`。

---

## 5. 任务执行状态机

```
              ┌──────────┐
              │ PENDING  │  TaskFrame 已提交，等待调度
              └────┬─────┘
                   │ 开始调度
                   ↓
           ┌──────────────┐
           │  PREFLIGHT   │  资源预检中（preflight=true 时）
           └──────┬───────┘
                  │ 预检通过
                  ↓
              ┌──────────┐
              │ RUNNING  │  子任务执行中
              └────┬─────┘
                   │
         ┌─────────┼─────────┐
         ↓         ↓         ↓
  ┌────────────┐ ┌──────┐ ┌──────────┐
  │WAITING_SYNC│ │FAILED│ │CANCELLED │
  │（同步屏障）│ └──┬───┘ └──────────┘
  └──────┬─────┘   │ 超过 max_retries
         │依赖完成  │ Orchestrator 收到 FAILED 通知
         ↓         ↓
    ┌───────────┐
    │ COMPLETED │
    └───────────┘
```

子任务状态：`PENDING` → `RUNNING` → `COMPLETED` / `FAILED` / `CANCELLED` / `SKIPPED`

### 5.1 重试语义

- 按节点 `retry_policy` 策略执行（默认指数退避）
- 重试时 MUST 使用相同的 `subtask_id` 和 `idempotency_key`
- 超过 `max_retries` 后，标记本子任务为 FAILED
- 子任务 FAILED 是否导致整体 FAILED，取决于 SyncFrame `min_required`：
  - 若失败数 > N - min_required，整体 FAILED
  - 否则（K-of-N 已满足），整体继续执行

### 5.2 任务取消

任意时刻，Orchestrator 可通过以下方式取消任务：

1. **直接断开连接**：Worker Agent MUST 检测到连接断开并停止执行
2. **发送取消 DelegateFrame**：`action="cancel"`, `params: { "task_id": "...", "subtask_id": "..." }`
3. **调用 NWP `system.task.cancel`**（若节点支持）

Worker Agent 收到取消信号后，MUST 停止执行并返回 `AlignStream(is_final=true, error.code="NOP-TASK-CANCELLED")`。

---

## 6. 多 Agent 协作完整流程

```
Orchestrator                              Worker B（数据）  Worker C（推理）
  │                                           │                   │
  │── TaskFrame(preflight=true) ─────────→   │                   │
  │      DelegateFrame(preflight) ─────────→ │                   │
  │      DelegateFrame(preflight) ────────────────────────────→  │
  │  ←── CapsFrame(available=true) ───────── │                   │
  │  ←── CapsFrame(available=true) ────────────────────────────  │
  │                                           │                   │
  │── DelegateFrame(fetch-data) ───────────→ │                   │
  │      Worker B 调用 NWP QueryFrame         │                   │
  │  ←── AlignStream(seq=0, data=products) ─ │                   │
  │  ←── AlignStream(is_final=true) ──────── │                   │
  │                                           │                   │
  │── DelegateFrame(analyze, products) ───────────────────────→  │
  │      Worker C 调用推理节点                                      │
  │  ←── AlignStream(seq=0, progress=0.5) ─────────────────────  │
  │  ←── AlignStream(is_final=true, result) ───────────────────  │
  │                                           │                   │
  │── SyncFrame(wait_for=[fetch,analyze])     │                   │
  │   sync 通过，aggregated result 已就绪     │                   │
  │                                           │                   │
  │  → callback_url POST（任务完成通知）      │                   │
```

---

## 7. 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NOP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | task_id 不存在 |
| `NOP-TASK-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 整体任务超时 |
| `NOP-TASK-DAG-INVALID` | `NPS-CLIENT-BAD-FRAME` | DAG 格式不合法（缺少起点/终点、字段错误等）|
| `NOP-TASK-DAG-CYCLE` | `NPS-CLIENT-BAD-FRAME` | DAG 中存在环路 |
| `NOP-TASK-DAG-TOO-LARGE` | `NPS-CLIENT-BAD-FRAME` | DAG 节点数超过上限（默认 32）|
| `NOP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | 任务已完成，无法重复提交 |
| `NOP-TASK-CANCELLED` | `NPS-CLIENT-CONFLICT` | 任务已被取消 |
| `NOP-DELEGATE-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | delegated_scope 超出父 Agent scope |
| `NOP-DELEGATE-REJECTED` | `NPS-CLIENT-UNPROCESSABLE` | Worker Agent 拒绝接受委托（能力不足或超负载）|
| `NOP-DELEGATE-CHAIN-TOO-DEEP` | `NPS-CLIENT-BAD-PARAM` | 委托链深度超过上限（默认 3 层）|
| `NOP-DELEGATE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 子任务在 deadline_at 前未完成 |
| `NOP-SYNC-TIMEOUT` | `NPS-SERVER-TIMEOUT` | SyncFrame 等待依赖任务超时 |
| `NOP-SYNC-DEPENDENCY-FAILED` | `NPS-CLIENT-UNPROCESSABLE` | 等待的依赖子任务已失败（且失败数超出 K-of-N 容忍范围）|
| `NOP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | AlignStream 序号不连续 |
| `NOP-STREAM-NID-MISMATCH` | `NPS-AUTH-UNAUTHENTICATED` | AlignStream sender_nid 与连接身份不一致 |
| `NOP-RESOURCE-INSUFFICIENT` | `NPS-SERVER-UNAVAILABLE` | 预检发现一个或多个 Worker Agent 资源不足（NPT / capability）|
| `NOP-CONDITION-EVAL-ERROR` | `NPS-CLIENT-BAD-PARAM` | DAG 节点 condition 表达式求值失败（语法错误或引用了不存在的字段）|
| `NOP-INPUT-MAPPING-ERROR` | `NPS-CLIENT-UNPROCESSABLE` | input_mapping JSONPath 无法解析或目标字段不存在 |

---

## 8. 安全考量

### 8.1 任务注入防御
Orchestrator MUST 验证接收到的 TaskFrame 来自可信 NID（通过 NIP 证书验证）。Worker Agent SHOULD 只接受已通过 NIP 验证的 DelegateFrame，拒绝未认证或 scope 不匹配的委托请求。

### 8.2 DAG 资源限制
实现 MUST 限制：
- DAG 最大节点数：**32**
- 最大委托链深度：**3 层**
- `condition` 表达式长度：**≤ 512 字符**
- `input_mapping` JSONPath 嵌套深度：**≤ 8 层**

### 8.3 审计追踪
每个 DelegateFrame 的执行记录 SHOULD 写入审计日志，包含：

- `sender_nid`（Orchestrator）
- `target_agent_nid`（Worker）
- `subtask_id`
- `parent_task_id`
- `timestamp`
- `trace_id`（来自 context，用于与 OpenTelemetry 系统关联）

### 8.4 callback_url 防滥用
- TaskFrame `callback_url` MUST 为 `https://` 前缀
- Orchestrator SHOULD 对回调 URL 做 SSRF 检查（禁止内网地址）
- 回调推送失败时 SHOULD 指数退避重试（最多 3 次），之后放弃并记录日志

### 8.5 Scope 委托链安全
每一层委托都必须通过 NIP CA 验证 `delegated_scope` 不超出父 scope。不得绕过 CA 直接委托超出权限的子任务。

---

## 9. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.3 | 2026-04-14 | DAG 节点粒度增强（per-node timeout/retry_policy/condition/input_mapping）；§3.1.2 context 字段支持 OpenTelemetry W3C Trace（trace_id/span_id/trace_flags/baggage）；§3.1.3 input_mapping JSONPath 映射；§3.1.4 retry_policy（fixed/linear/exponential）；§3.1.5 condition CEL 子集；DelegateFrame 新增 idempotency_key/priority/context/node_id；SyncFrame 新增 min_required（K-of-N 语义）和 §3.3.1/§3.3.2 聚合策略；AlignStream 新增 subtask_id/error 字段，§3.4.1 Token 级背压；§4 资源预检（preflight）协议；§5 扩展状态机（PREFLIGHT/SKIPPED）和任务取消机制；§6 完整多 Agent 流程图；3 个新错误码（RESOURCE-INSUFFICIENT、CONDITION-EVAL-ERROR、INPUT-MAPPING-ERROR、DELEGATE-TIMEOUT、TASK-CANCELLED）；§8.4 callback_url 防滥用；Depends-On 更新至 NCP v0.5 / NWP v0.5 |
| 0.2 | 2026-04-12 | 统一端口 17433；错误码改用 NPS 状态码映射；完善错误码列表 |
| 0.1 | 2026-04-10 | 初始规范：TaskFrame/DelegateFrame/SyncFrame/AlignStream，DAG 执行模型，替代 NCP AlignFrame |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
