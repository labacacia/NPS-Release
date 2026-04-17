[English Version](./NPS-2-NWP.en.md) | 中文版

# NPS-2: Neural Web Protocol (NWP)

**Spec Number**: NPS-2  
**Status**: Draft  
**Version**: 0.4  
**Date**: 2026-04-14  
**Port**: 17433（默认，共用）/ 17434（可选独立）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.4)  

> 本文档为 NWP 详细规范。套件总览见 [NPS-0-Overview.md](NPS-0-Overview.md)。

---

## 1. Terminology

本文档中的关键字 "MUST"、"MUST NOT"、"REQUIRED"、"SHOULD"、"MAY" 按照 RFC 2119 解释。

---

## 2. 协议概述

NWP 定义 AI Agent 访问 Web 数据和服务的方式。Agent 通过 `nwp://` 地址访问三类神经节点（Memory / Action / Complex），节点响应直接可被模型理解，无需任何语义解析层。

### 2.1 节点类型

| 类型 | 职责 | 典型数据源 |
|------|------|-----------|
| **Memory Node** | 数据存储与检索，不含计算逻辑 | RDS、NoSQL、文件系统、向量数据库 |
| **Action Node** | 执行操作，返回结果或副作用 | 函数、外部 API、消息队列、Webhook |
| **Complex Node** | 混合数据与操作，含子节点引用 | 以上所有类型 + 子节点引用 |
| **Gateway Node** | 无状态服务入口，将请求路由到 NOP 编排层 | AaaS 平台、多 Agent 服务网关 |

> **Gateway Node** 是 AaaS Profile 引入的扩展节点类型（见 `spec/services/NPS-AaaS-Profile.md`）。
> 它本身不执行业务逻辑，所有 Action 调用均转换为 NOP TaskFrame 下发给内部 Worker。
> 与 Complex Node 的关键区别：Gateway Node **无业务状态**，可无感知水平扩展。

### 2.2 Overlay 模式

在现有 HTTP 服务上附加 NWP 接口，服务器根据请求头区分访问者：

```
请求含 X-NWP-Agent 或 HelloFrame  →  返回 application/nwp-*
普通浏览器请求（无以上标识）        →  返回 text/html（正常网站）
```

Overlay 模式下 NWP 使用 HTTP 传输，帧序列化在 HTTP Body 中。详见 [NPS-1-NCP.md §2.2](NPS-1-NCP.md#22-传输模式)。

---

## 3. 节点地址规范

### 3.1 nwp:// URL 语法（ABNF）

```abnf
nwp-url     = "nwp://" host [":" port] "/" node-path ["/" sub-path]
host        = <RFC 3986 host>
port        = 1*DIGIT               ; 默认 17433
node-path   = segment *("/" segment)
sub-path    = "query" / "stream" / "invoke" / "subscribe" / "actions"
            / ".schema" / ".nwm"
segment     = 1*(ALPHA / DIGIT / "-" / "_")
```

### 3.2 子路径约定

| 子路径 | 方法 | 适用节点 | 描述 |
|--------|------|---------|------|
| `/query` | POST | Memory | 单次结构化查询（返回 CapsFrame）|
| `/stream` | POST | Memory | 流式查询（返回 StreamFrame 序列）|
| `/invoke` | POST | Action / Complex | 操作调用入口 |
| `/subscribe` | POST | Memory | 变更订阅入口（HTTP 模式，SSE）|
| `/actions` | GET | Action / Complex | 列举节点可调用操作（返回 NWM actions 子集 JSON）|
| `/.schema` | GET | 所有 | Schema 定义（返回 AnchorFrame JSON）|
| `/.nwm` | GET | 所有 | 完整节点清单（返回 NWM JSON）|

---

## 4. Neural Web Manifest (NWM)

每个节点 MUST 在 `/.nwm` 路径暴露机器可读清单，MIME 类型：`application/nwp-manifest+json`。

### 4.1 完整字段定义

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `nwp` | string | 必填 | NWP 版本，当前为 `"0.4"` |
| `node_id` | string | 必填 | 节点 NID，格式：`urn:nps:node:{host}:{path}` |
| `node_type` | string | 必填 | `"memory"` / `"action"` / `"complex"` / `"gateway"` |
| `display_name` | string | 可选 | 人类可读节点名称 |
| `manifest_version` | string | 可选 | 清单版本标识（ETag），用于条件请求缓存控制 |
| `min_agent_version` | string | 可选 | Agent 支持的最低 NPS 版本，格式 `"major.minor"`；低于此版本的 Agent MUST 被拒绝并返回 `NWP-MANIFEST-VERSION-UNSUPPORTED` |
| `wire_formats` | array | 必填 | 支持的编码格式列表：`["ncp-capsule", "msgpack", "json"]` |
| `preferred_format` | string | 必填 | 首选格式 |
| `schema_anchors` | object | 可选 | 预声明的 Schema 锚点，`{name: anchor_id}` |
| `capabilities` | object | 必填 | 节点能力声明，见 §4.2 |
| `data_sources` | array | 可选 | 底层数据源标识列表 |
| `auth` | object | 必填 | 认证要求，见 §4.3 |
| `rate_limits` | object | 可选 | 频率限制声明，见 §4.4 |
| `actions` | object | 条件必填 | Action/Complex/Gateway 节点 MUST 填写；操作注册表，见 §4.6 |
| `endpoints` | object | 必填 | 各功能端点 URL |
| `graph` | object | 可选 | 子节点引用（Complex Node 专用），见 §11 |
| `tokenizer_support` | array | 可选 | 节点支持的 tokenizer 列表（见 [token-budget.md](token-budget.md)）|

### 4.2 capabilities 字段

| 能力键 | 类型 | 描述 |
|--------|------|------|
| `query` | bool | 支持 QueryFrame（单次查询）|
| `stream_query` | bool | 支持流式查询（StreamFrame 响应）|
| `aggregate` | bool | 支持聚合查询（QueryFrame.aggregate）|
| `subscribe` | bool | 支持变更订阅（DiffFrame 推送）|
| `subscribe_filter` | bool | 订阅时支持携带 filter 条件 |
| `vector_search` | bool | 支持向量相似搜索 |
| `token_budget_hint` | bool | 支持根据 NPT 预算裁剪响应 |
| `ext_frame` | bool | 支持扩展帧头（大帧模式）|
| `e2e_enc` | bool | 支持 NCP E2E 加密（ENC=1，见 NPS-1-NCP §7.4）|
| `inline_anchor` | bool | 支持在响应中内联返回更新后的 AnchorFrame |

### 4.3 auth 字段

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `required` | bool | 必填 | 是否要求身份认证 |
| `identity_type` | string | 条件必填 | `"nip-cert"` / `"bearer"` / `"none"` |
| `trusted_issuers` | array | 条件必填 | 受信任的 CA URL 列表（identity_type 为 nip-cert 时必填）|
| `required_capabilities` | array | 可选 | Agent MUST 持有的能力列表，如 `["nwp:query"]` |
| `scope_check` | string | 可选 | scope 校验模式：`"prefix"`（默认）/ `"exact"` |
| `ocsp_url` | string | 可选 | OCSP 验证端点 |

### 4.4 rate_limits 字段

| 字段 | 类型 | 描述 |
|------|------|------|
| `requests_per_minute` | uint32 | 每 Agent 每分钟最大请求数 |
| `requests_per_day` | uint32 | 每 Agent 每日最大请求数 |
| `max_concurrent_streams` | uint32 | 每 Agent 最大并发流数 |
| `max_subscriptions` | uint32 | 每 Agent 最大并发订阅数 |

### 4.5 NWM 完整示例

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

**NWM 条件请求**

Agent SHOULD 缓存 NWM 并利用 `manifest_version` 做条件请求：HTTP 模式通过 `If-None-Match: {manifest_version}` 请求头，服务端若清单未变更则返回 `304 Not Modified`。

### 4.6 NWM Action 注册表

`actions` 字段为 `{action_id: ActionSpec}` 字典，Action/Complex Node MUST 在此声明所有可调用操作。

**ActionSpec 字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `description` | string | 可选 | 操作的人类可读说明 |
| `params_anchor` | string | 可选 | 参数 Schema 的 anchor_id（Agent 用于验证 ActionFrame.params）|
| `result_anchor` | string | 可选 | 结果 Schema 的 anchor_id（成功时 CapsFrame 使用此 anchor_ref）|
| `async` | bool | 必填 | 是否支持异步执行（true 时可在 ActionFrame 中设 async=true）|
| `idempotent` | bool | 可选 | 操作是否幂等（true 时 Agent 可安全重试）|
| `timeout_ms_default` | uint32 | 可选 | 默认超时毫秒数 |
| `timeout_ms_max` | uint32 | 可选 | 最大允许超时毫秒数 |
| `required_capability` | string | 可选 | 调用此操作所需的 NIP 能力，如 `"nwp:invoke"` |

**`/actions` 端点**

Agent 发起 `GET /actions`，节点返回 NWM `actions` 字段的完整 JSON（便于动态发现，无需下载整个 NWM）：

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

## 5. Schema 获取流程

Agent 通过以下流程获取 Node 的 Schema（AnchorFrame 由 Node 发布，Agent 只读引用）：

```
Agent                              Node
  │                                  │
  │── GET /.nwm ─────────────────→   │  1. 读取清单，获取 schema_anchors
  │←── NWM JSON ──────────────────   │     { "order": "sha256:a3f9..." }
  │                                  │
  │── GET /.schema ──────────────→   │  2. 获取完整 AnchorFrame（按需）
  │←── AnchorFrame JSON ──────────   │     Agent 本地缓存
  │                                  │
  │── QueryFrame(anchor_ref) ────→   │  3. 查询只携带 anchor_ref
  │←── CapsFrame(anchor_ref) ─────   │
```

Agent SHOULD 在首次连接时预加载 NWM 中声明的所有 schema_anchors 对应的 AnchorFrame，减少后续请求延迟。

---

## 6. QueryFrame (0x10)

用于 Memory Node 的结构化数据查询。

### 6.1 字段定义

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x10` |
| `anchor_ref` | string | 条件必填 | Schema anchor_id；聚合查询可省略 |
| `auto_anchor` | bool | 可选 | true 时若 anchor 过期，Node 在响应中自动附加最新 AnchorFrame，默认 true |
| `stream` | bool | 可选 | true 时触发流式查询模式（见 §6.6），响应为 StreamFrame 序列而非 CapsFrame |
| `aggregate` | object | 可选 | 聚合操作（见 §6.7）；设置后 `anchor_ref` 可省略 |
| `filter` | object | 可选 | 过滤条件，见 §6.2 |
| `fields` | array | 可选 | 返回字段列表；省略表示返回全部字段 |
| `limit` | uint32 | 可选 | 最大返回条数，默认 20，最大 1000；流式查询时为每帧最大条数 |
| `cursor` | string | 可选 | 分页游标，来自上一响应的 `next_cursor` |
| `order` | array | 可选 | 排序规则，见 §6.3 |
| `vector_search` | object | 可选 | 向量相似搜索，见 §6.4 |
| `token_budget` | uint32 | 可选 | NPT 预算上限（原生模式等价于 `X-NWP-Budget`）|
| `tokenizer` | string | 可选 | 使用的 tokenizer 标识（原生模式等价于 `X-NWP-Tokenizer`）|
| `depth` | uint8 | 可选 | 节点图谱遍历深度，默认 1，最大 5（原生模式等价于 `X-NWP-Depth`）|
| `request_id` | string | 可选 | UUID v4，用于请求追踪；节点在响应和日志中原样回传 |

### 6.2 Filter 语法

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `$eq` | 等于 | `{ "status": { "$eq": "active" } }` |
| `$ne` | 不等于 | `{ "status": { "$ne": "deleted" } }` |
| `$lt` | 小于 | `{ "price": { "$lt": 500 } }` |
| `$lte` | 小于等于 | `{ "price": { "$lte": 500 } }` |
| `$gt` | 大于 | `{ "stock": { "$gt": 0 } }` |
| `$gte` | 大于等于 | `{ "rating": { "$gte": 4.0 } }` |
| `$in` | 在列表中 | `{ "category": { "$in": ["phone", "tablet"] } }` |
| `$nin` | 不在列表中 | `{ "tag": { "$nin": ["discontinued"] } }` |
| `$contains` | 字符串包含（大小写敏感）| `{ "name": { "$contains": "Pro" } }` |
| `$between` | 范围（含两端）| `{ "price": { "$between": [100, 500] } }` |
| `$exists` | 字段是否存在 | `{ "thumbnail": { "$exists": true } }` |
| `$regex` | 正则匹配（UTF-8）| `{ "sku": { "$regex": "^PROD-[0-9]{4}$" } }` |
| `$and` | 逻辑与 | `{ "$and": [ {...}, {...} ] }` |
| `$or` | 逻辑或 | `{ "$or": [ {...}, {...} ] }` |
| `$not` | 逻辑非 | `{ "$not": { "status": { "$eq": "deleted" } } }` |

**`$regex` 安全约束**：模式长度 ≤ 256 字符；禁止嵌套量词（如 `(a+)+`）；节点 MUST 做 ReDoS 检测，违规返回 `NWP-QUERY-REGEX-UNSAFE`。

Filter 嵌套深度 MUST ≤ 8 层。

### 6.3 排序规则

```json
{ "order": [{ "field": "price", "dir": "ASC" }, { "field": "name", "dir": "ASC" }] }
```

### 6.4 向量搜索

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

支持 `metric`：`cosine`（默认）、`euclidean`、`dot_product`。节点通过 `capabilities.vector_search=true` 声明，不支持时返回 `NWP-QUERY-VECTOR-UNSUPPORTED`。

### 6.5 单次查询完整示例

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

### 6.6 流式查询协议

当 QueryFrame 中 `stream: true`（或使用 `/stream` 子路径）时，节点以 **StreamFrame (0x03) 序列**响应，而非单个 CapsFrame。需要节点 `capabilities.stream_query=true`。

**流式查询流程**

```
Agent                              Node
  │                                  │
  │── QueryFrame(stream:true) ────→  │
  │                                  │  分批查询，每批 limit 条
  │  ←── StreamFrame(seq=0) ───────  │  首帧，含 anchor_ref 和 estimated_total
  │  ←── StreamFrame(seq=1) ───────  │  后续帧，data 为下一批记录
  │       ...                        │
  │  ←── StreamFrame(is_last=true) ─ │  最终帧，is_last=true，data 可为空
```

**首帧附加字段（StreamFrame 扩展）**

流式查询时，首帧（seq=0）的 `data` 数组前 SHOULD 携带元数据帧头（通过 StreamFrame `error_code` 字段的对称扩展定义，或在节点实现中附加 `meta` 字段）：

| 字段 | 类型 | 描述 |
|------|------|------|
| `estimated_total` | uint64 | 符合 filter 条件的记录总估算数；-1 表示未知 |
| `request_id` | string | 回传 QueryFrame 中的 request_id |

**分页与流的关系**

- 流式查询不使用 `cursor`，记录按 `order` 连续推送，直到满足 `limit × 帧数` 或全量推送完毕
- Agent 如需提前终止，发送 SubscribeFrame（`action="unsubscribe"`, `stream_id` 等于 QueryFrame 的 `request_id`）或直接断开连接
- 节点 MUST 在连接断开后停止推送

### 6.7 聚合查询

当 QueryFrame 中包含 `aggregate` 字段时，节点返回聚合结果而非原始记录。需要节点 `capabilities.aggregate=true`。

**aggregate 字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `operations` | array | 必填 | 聚合操作列表，见下 |
| `group_by` | array | 可选 | 分组字段列表，如 `["category", "status"]` |
| `having` | object | 可选 | 分组后过滤（与 filter 语法相同，但字段名为 alias）|

**operation 元素**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `func` | string | 必填 | `COUNT` / `SUM` / `AVG` / `MIN` / `MAX` / `COUNT_DISTINCT` |
| `field` | string | 条件必填 | 聚合字段（`COUNT` 可省略，表示行数）|
| `alias` | string | 必填 | 结果字段名 |

**聚合查询示例**

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

**聚合响应（CapsFrame）**

聚合响应不使用业务 schema，`anchor_ref` 固定为 `"nps:system:aggregate:result"`：

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

用于 Action Node 和 Complex Node 的操作调用。

### 7.1 字段定义

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x11` |
| `action_id` | string | 必填 | 操作标识符，格式：`{domain}.{verb}`；系统保留操作见 §7.3 |
| `params` | object | 可选 | 操作参数，Schema 由 NWM actions.{action_id}.params_anchor 定义 |
| `idempotency_key` | string | 可选 | 幂等键（UUID v4），有效期 24 小时 |
| `timeout_ms` | uint32 | 可选 | 超时毫秒数，默认 5000，最大 300000 |
| `async` | bool | 可选 | true 表示异步执行，响应返回 `task_id` |
| `callback_url` | string | 可选 | 异步任务完成时的回调 URL（`https://`）|
| `priority` | string | 可选 | 任务优先级：`"low"` / `"normal"`（默认）/ `"high"` |
| `request_id` | string | 可选 | UUID v4，用于请求追踪（回传至响应和 task status）|

### 7.2 异步任务状态机

```
PENDING → RUNNING → COMPLETED
                  ↘ FAILED
                  ↘ CANCELLED
```

异步执行时，初始响应（CapsFrame）：

```json
{
  "task_id": "uuid-v4",
  "status": "pending",
  "poll_url": "nwp://api.example.com/orders/actions/status/uuid-v4",
  "estimated_ms": 3000,
  "request_id": "550e8400-..."
}
```

### 7.3 系统保留操作

所有支持异步 Action 的节点 MUST 实现：

| action_id | 描述 | 必填参数 | 响应 |
|-----------|------|---------|------|
| `system.task.status` | 轮询任务状态 | `{ "task_id": "uuid" }` | 任务状态对象（见下）|
| `system.task.cancel` | 取消任务 | `{ "task_id": "uuid" }` | `{ "cancelled": true }` 或错误 |

**`system.task.status` 响应**

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

### 7.4 完整示例

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

用于在 Memory Node 上建立变更订阅，服务端以 DiffFrame (0x02) 推送增量更新。

### 8.1 字段定义

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x12` |
| `action` | string | 必填 | `"subscribe"` / `"unsubscribe"` / `"ping"` |
| `stream_id` | string | 必填 | 客户端生成的 UUID v4 |
| `anchor_ref` | string | 条件必填 | 订阅数据的 anchor_id（`action="subscribe"` 时必填）|
| `filter` | object | 可选 | 过滤条件（同 §6.2）；需 `capabilities.subscribe_filter=true` |
| `heartbeat_interval` | uint32 | 可选 | 心跳间隔秒数，0 = 禁用，默认 30 |
| `resume_from_seq` | uint64 | 可选 | 断线恢复时提供，从此序号之后重播事件（含该序号对应事件）；节点无法满足时返回 `NWP-SUBSCRIBE-SEQ-TOO-OLD` |

### 8.2 DiffFrame 订阅扩展字段

订阅推送的 DiffFrame（0x02）在标准字段基础上附加：

| 字段 | 类型 | 描述 |
|------|------|------|
| `stream_id` | string | 关联的订阅流 ID |
| `seq` | uint64 | 单调递增的事件序号（per-stream，从 1 开始）；Agent 用于检测丢帧和断线恢复 |
| `event_type` | string | `"create"` / `"update"` / `"delete"` |
| `timestamp` | string | 变更发生时间（ISO 8601）|

**序号语义**

- 每个 stream_id 的 seq 独立维护，从 1 开始，单调递增
- Agent 检测到 seq 不连续时，SHOULD 使用 `resume_from_seq` 重新订阅
- 节点 SHOULD 缓冲最近 N 个事件（推荐缓冲 10 分钟或 10,000 条，取先达者）
- 若 `resume_from_seq` 超出缓冲范围，节点 MUST 返回 `NWP-SUBSCRIBE-SEQ-TOO-OLD`（Agent 需全量重查后重新订阅）

### 8.3 订阅流程

```
Agent                                  Node
  │                                       │
  │── SubscribeFrame(subscribe, sid) ───→ │  建立订阅
  │  ←─────────────────── CapsFrame(ack) │  确认（stream_id, status="subscribed"）
  │                                       │
  │  ←─── DiffFrame(seq=1, event) ──────  │  变更推送
  │  ←─── DiffFrame(seq=2, event) ──────  │
  │  ←─── DiffFrame(seq=N, ping) ───────  │  心跳（patch=[]）
  │                                       │
  │  [ 连接中断 ]                          │
  │── SubscribeFrame(subscribe, sid,      │  断线恢复
  │      resume_from_seq=N) ───────────→  │
  │  ←─────────────────── CapsFrame(ack) │  确认（status="subscribed", resumed=true）
  │  ←─── DiffFrame(seq=N+1, ...) ──────  │  从断点续播
```

**订阅确认 CapsFrame**

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

- `last_seq`：节点当前已知的最新事件 seq（新订阅时为 0，恢复时为缓冲中最新事件的 seq）
- `resumed`：true 表示这是一次断线恢复

### 8.4 HTTP 模式下的订阅（SSE）

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

## 9. HTTP 头（HTTP 模式）

### 9.1 请求头

| 头 | 必填 | 描述 |
|----|------|------|
| `X-NWP-Agent` | 条件必填 | Agent NID（auth.required=true 时必填）|
| `X-NWP-Budget` | 可选 | NPT 预算上限（uint32）|
| `X-NWP-Tokenizer` | 可选 | Agent 使用的 tokenizer 标识 |
| `X-NWP-Depth` | 可选 | 节点图谱遍历深度，默认 1，最大 5 |
| `X-NWP-Encoding` | 可选 | 请求编码 Tier：`json`/`msgpack`，默认 `msgpack` |
| `X-NWP-Request-ID` | 可选 | UUID v4，请求追踪 ID；节点在响应头中原样回传 |
| `If-None-Match` | 可选 | NWM 条件请求；值为 `manifest_version` |
| `Content-Type` | 必填 | `application/nwp-frame` |

### 9.2 响应头

| 头 | 描述 |
|----|------|
| `X-NWP-Schema` | 响应使用的 anchor_id |
| `X-NWP-Tokens` | 实际 NPT 消耗 |
| `X-NWP-Tokens-Native` | 原生 token 消耗 |
| `X-NWP-Tokenizer-Used` | 实际使用的 tokenizer |
| `X-NWP-Cached` | `true` 表示命中缓存 |
| `X-NWP-Node-Type` | 节点类型 |
| `X-NWP-Request-ID` | 回传请求方的 `X-NWP-Request-ID`（若未提供，节点 MAY 自动生成）|
| `X-NWP-Rate-Limit` | 每分钟请求上限 |
| `X-NWP-Rate-Remaining` | 本分钟剩余请求数 |
| `X-NWP-Rate-Reset` | 限速窗口重置时间（Unix 时间戳）|
| `Content-Type` | `application/nwp-capsule`（正常响应）/ `application/nwp-error+json`（错误响应）|

### 9.3 原生模式字段映射

| HTTP 头 | QueryFrame 字段 | ActionFrame 字段 |
|---------|----------------|-----------------|
| `X-NWP-Agent` | — (HelloFrame `agent_id` 握手时已声明) | 同左 |
| `X-NWP-Budget` | `token_budget` | — |
| `X-NWP-Tokenizer` | `tokenizer` | — |
| `X-NWP-Depth` | `depth` | — |
| `X-NWP-Request-ID` | `request_id` | `request_id` |

### 9.4 HTTP 模式错误响应格式

HTTP 模式下，错误响应使用以下格式，`Content-Type: application/nwp-error+json`：

```json
{
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NWP-ACTION-NOT-FOUND",
  "message": "Action 'orders.ship' is not registered on this node",
  "details": { "action_id": "orders.ship" },
  "request_id": "550e8400-e29b-41d4-a716-446655440003"
}
```

HTTP 状态码由 NPS 状态码映射决定，见 [status-codes.md](status-codes.md)。

**字段说明**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `status` | string | 必填 | NPS 状态码 |
| `error` | string | 必填 | 协议级错误码（如 `NWP-ACTION-NOT-FOUND`）|
| `message` | string | 可选 | 人类可读描述 |
| `details` | object | 可选 | 结构化错误附加信息 |
| `request_id` | string | 可选 | 回传请求中的 `X-NWP-Request-ID` |

---

## 10. 完整请求响应示例

**HTTP 模式查询请求**

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

**成功响应**

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

**错误响应**

```
HTTP/1.1 404 Not Found
X-NWP-Request-ID: 550e8400-e29b-41d4-a716-446655440001
Content-Type: application/nwp-error+json

{ "status": "NPS-CLIENT-NOT-FOUND", "error": "NWP-QUERY-FIELD-UNKNOWN", ... }
```

---

## 11. Complex Node — 节点图谱

Complex Node 在 NWM 中声明子节点引用：

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

Agent 通过 `X-NWP-Depth` 头（HTTP 模式）或 QueryFrame `depth` 字段（原生模式）控制遍历深度。节点 MUST 检测循环引用（返回 `NWP-GRAPH-CYCLE`），并维护子节点 URL 白名单（防 SSRF）。

---

## 12. 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NWP-AUTH-NID-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Agent scope 不覆盖目标节点 |
| `NWP-AUTH-NID-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | NID 证书已过期 |
| `NWP-AUTH-NID-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | NID 已被吊销 |
| `NWP-AUTH-NID-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | NID 颁发者不在 trusted_issuers 中 |
| `NWP-AUTH-NID-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Agent 缺少节点要求的能力 |
| `NWP-QUERY-FILTER-INVALID` | `NPS-CLIENT-BAD-PARAM` | Filter 语法不合法或嵌套超限 |
| `NWP-QUERY-FIELD-UNKNOWN` | `NPS-CLIENT-BAD-PARAM` | fields 中引用了不存在的字段 |
| `NWP-QUERY-CURSOR-INVALID` | `NPS-CLIENT-BAD-PARAM` | cursor 值无法解码或已过期 |
| `NWP-QUERY-REGEX-UNSAFE` | `NPS-CLIENT-BAD-PARAM` | `$regex` 模式被拒绝（ReDoS 风险或超长）|
| `NWP-QUERY-VECTOR-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持向量搜索 |
| `NWP-QUERY-AGGREGATE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持聚合查询 |
| `NWP-QUERY-AGGREGATE-INVALID` | `NPS-CLIENT-BAD-PARAM` | aggregate 结构不合法（未知 func、alias 重复等）|
| `NWP-QUERY-STREAM-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持流式查询 |
| `NWP-ACTION-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | action_id 不存在 |
| `NWP-ACTION-PARAMS-INVALID` | `NPS-CLIENT-UNPROCESSABLE` | 操作参数 Schema 校验失败 |
| `NWP-ACTION-IDEMPOTENCY-CONFLICT` | `NPS-CLIENT-CONFLICT` | 相同 idempotency_key 的请求正在进行中 |
| `NWP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | task_id 不存在 |
| `NWP-TASK-ALREADY-CANCELLED` | `NPS-CLIENT-CONFLICT` | 任务已被取消 |
| `NWP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | 任务已完成，无法取消 |
| `NWP-TASK-ALREADY-FAILED` | `NPS-CLIENT-CONFLICT` | 任务已失败，无法取消 |
| `NWP-SUBSCRIBE-STREAM-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | unsubscribe 的 stream_id 不存在 |
| `NWP-SUBSCRIBE-LIMIT-EXCEEDED` | `NPS-LIMIT-EXCEEDED` | 超出最大并发订阅数 |
| `NWP-SUBSCRIBE-FILTER-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | 节点不支持带 filter 的订阅 |
| `NWP-SUBSCRIBE-INTERRUPTED` | `NPS-SERVER-UNAVAILABLE` | 订阅流因数据源中断而终止 |
| `NWP-SUBSCRIBE-SEQ-TOO-OLD` | `NPS-CLIENT-CONFLICT` | resume_from_seq 超出节点缓冲范围，需全量重查 |
| `NWP-BUDGET-EXCEEDED` | `NPS-LIMIT-BUDGET` | 响应将超过 token 预算 |
| `NWP-DEPTH-EXCEEDED` | `NPS-CLIENT-BAD-PARAM` | depth 超过节点允许的 max_depth |
| `NWP-GRAPH-CYCLE` | `NPS-CLIENT-UNPROCESSABLE` | 节点图谱存在循环引用 |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | 底层数据源暂不可用 |
| `NWP-MANIFEST-VERSION-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Agent NPS 版本低于 min_agent_version |
| `NWP-RATE-LIMIT-EXCEEDED` | `NPS-LIMIT-RATE` | 超出频率限制 |

---

## 13. 安全考量

### 13.1 Scope 强制校验
节点 MUST 在每次请求时校验 Agent NID 的 scope。超出 scope 的请求 MUST 返回 `NWP-AUTH-NID-SCOPE-VIOLATION`，不得返回任何数据。

### 13.2 SSRF 防护
Complex Node 解析子节点引用时，MUST 维护允许的节点 URL 前缀白名单，禁止访问内网地址（RFC 1918）。

### 13.3 Token Budget 强制执行
超过预算时，节点 SHOULD 优先裁剪响应内容（字段精简 → 摘要 → 截断记录数）；无法裁剪时 MUST 返回 `NWP-BUDGET-EXCEEDED`，不得直接截断数据。详见 [token-budget.md §4.3](token-budget.md)。

### 13.4 频率限制
节点 SHOULD 对每个 Agent NID 实施频率限制。超限时返回 `NWP-RATE-LIMIT-EXCEEDED` 并附加 `X-NWP-Rate-Reset` 头。未认证请求 SHOULD 使用 IP 维度限制。

### 13.5 Filter 注入防护
- 字段名 MUST 仅含字母/数字/下划线/点，长度 ≤ 128 字符
- `$regex` MUST 经过 ReDoS 检测；Filter 嵌套深度 ≤ 8
- 节点 MUST 使用参数化查询，禁止字符串拼接

### 13.6 callback_url 防滥用
- ActionFrame `callback_url` MUST 为 `https://` 前缀
- 节点 SHOULD 对回调 URL 做 SSRF 检查（禁止内网地址）
- 节点 SHOULD 对 callback 推送失败做指数退避重试（最多 3 次），之后放弃并标记任务为 `COMPLETED` 而非无限重试

---

## 14. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.4 | 2026-04-14 | §3.2 新增 `/actions` 子路径；§4.1 NWM 新增 `actions` 字段；§4.2 capabilities 新增 stream_query、aggregate；§4.6 NWM Action 注册表（ActionSpec、params_anchor/result_anchor/async/idempotent）；QueryFrame §6.1 新增 `stream`、`aggregate`、`request_id`；§6.6 流式查询协议（StreamFrame 序列、estimated_total、提前终止）；§6.7 聚合查询（COUNT/SUM/AVG/MIN/MAX/COUNT_DISTINCT、group_by、having）；ActionFrame §7.1 新增 `request_id`；SubscribeFrame §8.1 新增 `resume_from_seq`；§8.2 DiffFrame 扩展字段（seq 单调递增、event_type、timestamp）及断线恢复语义；§9.1/9.2 新增 X-NWP-Request-ID；§9.4 HTTP 模式错误响应格式（application/nwp-error+json）；§10 更新完整示例（含错误响应）；§13.6 callback_url 防滥用安全节；新增 5 条错误码（AGGREGATE-UNSUPPORTED/-INVALID、STREAM-UNSUPPORTED、SUBSCRIBE-SEQ-TOO-OLD、task cancel 系列）|
| 0.3 | 2026-04-14 | SubscribeFrame (0x12)；auto_anchor；Filter $not/$exists/$regex；ActionFrame callback_url/priority；system.task.*；NWM min_agent_version/rate_limits；§13.4/13.5 安全节 |
| 0.2 | 2026-04-12 | 统一端口 17433；AnchorFrame 改为 Node 发布；NPT 计量；NPS 状态码映射 |
| 0.1 | 2026-04-10 | 初始规范 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
