[English Version](./NPS-2-NWP.md) | 中文版

# NPS-2: Neural Web Protocol (NWP)

**Spec Number**: NPS-2  
**Status**: Proposed  
**Version**: 0.8  
**Date**: 2026-04-27  
**Port**: 17433（默认，共用）/ 17434（可选独立）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.6)、NPS-3 (NIP v0.4)、NPS-4 (NDP v0.5)  

> 本文档为 NWP 详细规范。套件总览见 [NPS-0-Overview.cn.md](NPS-0-Overview.cn.md)。

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
| **Anchor Node** | 集群控制平面与对外入口 —— 把入站 NWP `Action`/`Query` 帧通过 NOP 路由给成员节点，可选维护成员节点拓扑 | AaaS 平台、多 Agent 服务网关、子集群路由器 |
| **Bridge Node** | 在 NPS 帧与非-NPS 协议（HTTP/HTTPS、gRPC、MCP、A2A）之间翻译 | 调用遗留 REST API、gRPC 服务、Model Context Protocol 服务端、Agent-to-Agent 端点 |

一个节点 MAY 同时承担多个角色（例如同一进程既是 Memory Node 又是 Anchor Node，无须分离时）。多角色声明放在 NDP `Announce` 帧的 `node_kind` 字段，该字段接受单个字符串或字符串数组（NPS-4 §3.1）。

> **Anchor Node** 与 **Bridge Node** 由 [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md) 一同引入，替换原 `Gateway Node` 类型：
> - **Anchor Node** 继承了 Gateway Node 原本承载的"集群入口 + NOP 路由"角色。它在每次请求中无状态，但 MAY 维持一份成员节点的长期注册表。
> - **Bridge Node** 是新类型，职责是 **NPS → 外部协议**翻译。每次请求无状态，不参与集群拓扑。（方向说明：这与 `compat/*-bridge` 历史上发布的"外部协议 → NPS"入站适配器方向相反；后者已重命名为 `compat/*-ingress`，把 "Bridge" 一词让出给本节点类型。）
> - 原 `Gateway Node` 术语已弃用；`"gateway"` wire 值被移除，解析器 MUST 拒绝并明确报告 CR-0001。

#### 已移除类型

> **Gateway Node**（v1.0-alpha.3 移除）—— 拆为 **Anchor Node**（集群入口 / NOP 路由）与 **Bridge Node**（NPS↔非-NPS 协议翻译）。完整背景与迁移说明见 [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md)。实现 MUST 拒绝遗留 `node_type: "gateway"` 与 `node_kind: "gateway"` wire 值；SDK SHOULD 在失败前一次性给出指向 CR 的弃用提示。

#### Anchor Node —— 详细语义

Anchor Node MUST：

1. 接受寻址到集群（即 Anchor 自身的 NID，而非具体成员 NID）的入站 NWP `Action` / `Query` 帧。
2. 根据成员节点声明的能力与当前负载，把帧分派给合适的成员。参考分派路径把 `ActionFrame` 转为 NOP `TaskFrame`，委托给本机 NOP orchestrator（见 [NPS-AaaS-Profile §2](services/NPS-AaaS-Profile.cn.md)）。
3. 把成员节点的出站响应聚合成单条响应流回送给原始调用方。
4. 可选地维护集群内成员节点的注册表（NID、声明能力、`activation_mode`，见 [NPS-Node Profile §6](services/NPS-Node-Profile.cn.md)）。成员节点入集群时通过 NDP `Announce` 帧携带 `cluster_anchor` 引用 Anchor Node 的 NID 完成注册；下线遵循标准 NDP 离线语义。

集群 MUST 至少有一个 Anchor Node。HA 部署 MAY 为同一集群运行多个 Anchor Node；Anchor Node 之间的共识协议是实现自定的，推迟到 NPS-AaaS Profile L3。

维护成员注册表的 Anchor Node MUST 通过保留查询类型 `topology.snapshot` 与 `topology.stream`（§12）暴露注册表内容，详见 [NPS-CR-0002](cr/NPS-CR-0002-anchor-topology-queries.md)。两者在 NPS-AaaS Profile L2 及以上等级强制要求。

#### Bridge Node —— 详细语义

Bridge Node MUST：

1. 接受携带 `bridge_target` 参数（标识外部协议与端点）的入站 NWP 帧。`bridge_target` 的具体 schema 在本 CR 中实现自定；按协议标准化推迟到后续 CR。
2. 用目标协议的格式产出对外请求。
3. 把目标协议的响应翻译回 NWP 帧（通常是 `CapsFrame`）。

Bridge Node 每次请求无状态，不参与集群拓扑。一个 Bridge Node MAY 翻译多个不同外部协议；部署 MAY 为隔离起见为每个协议跑独立 Bridge Node。

参考 Bridge Node 实现期望支持的标准外部协议：

- HTTP / HTTPS（REST 与 streaming）
- gRPC（unary 与 streaming）
- MCP（Model Context Protocol）
- A2A（Agent-to-Agent 协议）

更多协议适配器 MAY 通过未来 CR 注册。支持的协议集合在 NDP `Announce.bridge_protocols` 中声明（NPS-4 §3.1）。

### 2.2 Overlay 模式

在现有 HTTP 服务上附加 NWP 接口，服务器根据请求头区分访问者：

```
请求含 X-NWP-Agent 或 HelloFrame  →  返回 application/nwp-*
普通浏览器请求（无以上标识）        →  返回 text/html（正常网站）
```

Overlay 模式下 NWP 使用 HTTP 传输，帧序列化在 HTTP Body 中。详见 [NPS-1-NCP.cn.md §2.2](NPS-1-NCP.cn.md#22-传输模式)。

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
| `node_type` | string | 必填 | `"memory"` / `"action"` / `"complex"` / `"anchor"` / `"bridge"`。遗留值 `"gateway"` 在 v1.0-alpha.3 移除（见 §2.1 *已移除类型* 与 [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md)），解析器 MUST 拒绝。同时承担多角色的节点请在 NDP `Announce.node_kind`（NPS-4 §3.1）中声明 —— 此处 `node_type` 表示 `/.nwm` 上对外暴露的**主**角色。 |
| `display_name` | string | 可选 | 人类可读节点名称 |
| `manifest_version` | string | 可选 | 清单版本标识（ETag），用于条件请求缓存控制 |
| `min_agent_version` | string | 可选 | Agent 支持的最低 NPS 版本，格式 `"major.minor"`；低于此版本的 Agent MUST 被拒绝并返回 `NWP-MANIFEST-VERSION-UNSUPPORTED` |
| `min_assurance_level` | string | 可选 | 取值 `"anonymous"`（默认）/ `"attested"` / `"verified"` 之一（见 [NIP §5.1.1](NPS-3-NIP.cn.md#511-保证等级nps-rfc-0003)）。请求等级低于此值时 MUST 返回 `NWP-AUTH-ASSURANCE-TOO-LOW`（`NPS-AUTH-FORBIDDEN`），响应 SHOULD 在 `hint` 字段附 CA 注册 URL。默认 `"anonymous"` 与 v1.0-alpha.2 节点向后兼容。可在 §4.6 单个 ActionSpec 上以 `auth.min_assurance_level` 字段覆盖。（NPS-RFC-0003）|
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
| `tokenizer_support` | array | 可选 | 节点支持的 tokenizer 列表（见 [token-budget.cn.md](token-budget.cn.md)）|

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
| `type` | string | 可选 | 保留查询类型标识，见 §12。设置后按对应类型的字段语义解释，`anchor_ref` 由该类型重新定义；不设置则按下方默认 per-anchor 查询行为 |
| `anchor_ref` | string | 条件必填 | Schema anchor_id；聚合查询可省略；`type` 选中的保留类型若不需要可省略 |
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
| `type` | string | 可选 | 保留订阅类型标识，见 §12。设置后按对应类型的字段语义解释，`anchor_ref` 由该类型重新定义；不设置则按下方默认 per-anchor 订阅行为 |
| `stream_id` | string | 必填 | 客户端生成的 UUID v4 |
| `anchor_ref` | string | 条件必填 | 订阅数据的 anchor_id（`action="subscribe"` 且 `type` 缺省或保留类型要求 anchor 时必填）|
| `filter` | object | 可选 | 过滤条件（同 §6.2）；需 `capabilities.subscribe_filter=true` |
| `heartbeat_interval` | uint32 | 可选 | 心跳间隔秒数，0 = 禁用，默认 30 |
| `resume_from_seq` | uint64 | 可选 | 断线恢复时提供，从此序号之后重播事件（含该序号对应事件）；节点无法满足时返回 `NWP-SUBSCRIBE-SEQ-TOO-OLD` |

### 8.2 DiffFrame 订阅扩展字段

订阅推送的 DiffFrame（0x02）在标准字段基础上附加：

| 字段 | 类型 | 描述 |
|------|------|------|
| `stream_id` | string | 关联的订阅流 ID |
| `seq` | uint64 | 单调递增的事件序号（per-stream，从 1 开始）；Agent 用于检测丢帧和断线恢复 |
| `event_type` | string | 默认订阅取 `"create"` / `"update"` / `"delete"`。保留订阅类型（§12）MAY 定义额外取值（例如 §12.2 拓扑事件类型）|
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

HTTP 状态码由 NPS 状态码映射决定，见 [status-codes.cn.md](status-codes.cn.md)。

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

## 12. 保留查询类型

`QueryFrame`（§6.1）与 `SubscribeFrame`（§8.1）的 `type` 字段允许请求选入**保留查询类型**，由本规范定义其语义。`topology.*` 命名空间保留给 Anchor Node 上的集群拓扑操作；以下保留命名空间对所列节点角色**强制要求**。

| 命名空间 | 所属角色 | 强制等级 | 操作 |
|----------|---------|---------|------|
| `topology.*` | Anchor Node（§2.1）| NPS-AaaS Profile L2（[services/NPS-AaaS-Profile.cn.md §4.3](services/NPS-AaaS-Profile.cn.md)）| `topology.snapshot`（§12.1）、`topology.stream`（§12.2）|

`type` 缺省时按默认 per-anchor 查询/订阅语义（§6、§8）执行。`type` 设置后，本节定义的字段生效；与之冲突的标准字段（如 `anchor_ref`、顶层 `filter`）将被忽略，除非保留类型自身的 schema 显式带上它们。

实现遇到无法识别的保留 `type` 值时，MUST 用 `NWP-ACTION-NOT-FOUND`（QueryFrame）或 `NWP-SUBSCRIBE-FILTER-UNSUPPORTED`（SubscribeFrame）显式拒绝，让调用方决定是否回退到默认行为。

### 12.1 `topology.snapshot`

一次性获取 Anchor Node 的集群拓扑快照。

| 属性 | 取值 |
|------|------|
| 帧 | QueryFrame（0x10），`type = "topology.snapshot"` |
| 强制对象 | 全部 Anchor Node（NPS-AaaS Profile L2 及以上）|
| 幂等 | 是 |
| 缓存 | 响应 MAY 由客户端缓存；响应中的 `version` 字段用于与后续 `topology.stream` 事件做校验对齐 |

**请求字段**（QueryFrame 顶层，叠加在 §6.1 标准字段之上）：

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `type` | string | 必填 | 常量 `"topology.snapshot"` |
| `topology` | object | 必填 | 拓扑相关参数容器，见下 |
| `topology.scope` | string | 必填 | `"cluster"` 表示 Anchor 自身集群；`"member"` 表示单个成员（要求带 `topology.target_nid`）|
| `topology.include` | array of strings | 可选 | `["members", "capabilities", "tags", "metrics"]` 的子集，默认 `["members"]`。`capabilities` 与 `metrics` 的 schema 由实现自定，可为空 |
| `topology.depth` | uint8 | 可选 | 控制对子 Anchor 成员的递归。`1`（默认）只把子 Anchor 当引用列出；`2+` 递归展开。Anchor Node MAY 设上限并以 `truncated: true` 提示截断。在 L2，深度 ≥ 2 的子 Anchor 递归为 **可选**；客户端 SHOULD 自行对每个子 Anchor 分别发 snapshot |
| `topology.target_nid` | string | 条件必填 | 当 `topology.scope = "member"` 时必填，指目标成员的 NID |

**响应**：`CapsFrame (0x04)`，`anchor_ref = "nps:system:topology:snapshot"`，`data` 数组含一条快照对象。

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
        "node_kind": ["memory"],
        "activation_mode": "ephemeral",
        "tags": ["dev", "library"],
        "joined_at": "2026-04-15T10:23:00Z",
        "last_seen": "2026-04-26T14:55:00Z"
      },
      {
        "nid": "urn:nps:node:labacacia:host-def456",
        "node_kind": ["anchor"],
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

**Snapshot payload 字段**：

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `version` | uint64 | 必填 | 单调递增的拓扑版本号。仅在 Anchor 重启 / rebase 时重置（§12.3）|
| `anchor_nid` | string | 必填 | 响应方 Anchor Node 的 NID |
| `cluster_size` | uint32 | 必填 | 直接成员总数，与 `topology.depth` 截断无关 |
| `members` | array of member objects | 必填 | 见下方成员对象 schema |
| `truncated` | bool | 可选 | 当 `topology.depth` 上限被触发时为 true；否则省略或 false |

**Member 对象 schema**：

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `nid` | string | 必填 | 成员 NID |
| `node_kind` | array of strings | 必填 | NDP `node_kind` 取值（NPS-4 §3.1）|
| `activation_mode` | string | 必填 | NDP `activation_mode`，取 `ephemeral` / `resident` / `hybrid`（NPS-4）|
| `child_anchor` | bool | 可选 | 该成员自身是否为子集群的 Anchor；为真时隐含必带 `member_count` |
| `member_count` | uint32 | 条件必填 | `child_anchor = true` 时必填，表示子 Anchor 直接成员数 |
| `tags` | array of strings | 可选 | NDP 声明的 tag |
| `joined_at` | string | 可选 | RFC 3339 时间戳，首次观测时间 |
| `last_seen` | string | 可选 | RFC 3339 时间戳，最近一次 NDP `Announce` |
| `capabilities` | object | 可选 | 仅当 `topology.include` 包含 `capabilities` 时返回；schema 由实现自定 |
| `metrics` | object | 可选 | 仅当 `topology.include` 包含 `metrics` 时返回；schema 由实现自定 |

### 12.2 `topology.stream`

Anchor Node 集群拓扑的持续变更事件流。

| 属性 | 取值 |
|------|------|
| 帧 | SubscribeFrame（0x12），`type = "topology.stream"` |
| 强制对象 | 全部 Anchor Node（NPS-AaaS Profile L2 及以上）|
| 可取消 | 是，使用标准 `SubscribeFrame.action = "unsubscribe"` |

**请求字段**（SubscribeFrame 顶层，叠加在 §8.1 标准字段之上）：

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `type` | string | 必填 | 常量 `"topology.stream"` |
| `topology` | object | 必填 | 拓扑相关参数容器，见下 |
| `topology.scope` | string | 必填 | `"cluster"`（默认指 Anchor 自身集群）；其它 scope 留待将来 |
| `topology.filter` | object | 可选 | 减少事件量。支持键：`tags_any`（数组，命中任一）、`tags_all`（数组，全部命中）、`node_kind`（数组）。Anchor Node MUST 以 `NWP-TOPOLOGY-FILTER-UNSUPPORTED` 拒绝未识别 key |
| `topology.since_version` | uint64 | 可选 | 从指定 version 之后恢复。Anchor Node MUST 在保留窗口内重播缺失事件；版本超出保留窗口时 MUST 推送 `resync_required` 事件，客户端 MUST 重发 `topology.snapshot` |

`type = "topology.stream"` 时，`topology.since_version` 是 `SubscribeFrame.resume_from_seq`（§8.1）在拓扑语境下的同义字段。两者同时出现时，`topology.since_version` 优先。

**事件**通过 `DiffFrame (0x02)` 推送，遵循 §8.2 扩展字段。`topology.stream` 上下文下 `event_type` 取下表保留拓扑事件类型（在默认 `"create" / "update" / "delete"` 之外扩展），`seq` 是事件后的拓扑版本（§12.3），`payload` 携带类型相关数据。

| `event_type` | 触发条件 | `payload` 形态 |
|--------------|---------|---------------|
| `member_joined` | NDP `Announce` 中 `cluster_anchor` 指向本 Anchor | 完整成员对象（§12.1）|
| `member_left` | 成员显式离开或超过 NDP 在线 TTL | `{ "nid": "urn:nps:..." }` |
| `member_updated` | 既有成员元数据变更（tags、activation_mode、capabilities 等）| `{ "nid": "urn:nps:...", "changes": { "<字段>": <新值>, ... } }` —— 仅字段级 diff，由客户端自行合并 |
| `anchor_state` | 罕见。Anchor Node 内部状态变更需通知客户端（如重启后版本计数 rebase）| `{ "field": "version_rebased", "details": { ... } }` |
| `resync_required` | 订阅方的 `topology.since_version` 已无法重播 | `{ "reason": "version_too_old" }`。本事件不带 `seq`，订阅方 MUST 重新发起 `topology.snapshot` |

标准 SubscribeFrame 心跳（§8.2）与取消订阅（§8.1，`action = "unsubscribe"`）继续生效。

### 12.3 版本与一致性模型

**保证**：

- `version: V` 的 `topology.snapshot` 反映恰好 `V` 次拓扑变更后的集群状态。
- `seq: V` 的 `topology.stream` 事件反映恰好 `V` 次变更后的集群状态。
- `version: V` 快照叠加之后 `V+1, V+2, …` 的事件序列，得到一致的实时视图。

**不保证**：

- 实时投递延迟 —— 事件 MAY 批量推送。
- 多个 Anchor Node 之间的事件全序 —— 每个 Anchor 维护独立 `version` 计数器。
- 与非拓扑事件（成员节点上的 Action / Query 流量）的全序。

**重启与 rebase**：Anchor Node MAY 在重启时 rebase 自己的 `version` 计数器。rebase 时 MUST 向所有活跃订阅推送 `anchor_state` 事件并附 `field: "version_rebased"`；订阅方 MUST 视同 `resync_required` 处理，重新发起 `topology.snapshot`。

### 12.4 不在范围内

- **Capability / metrics 字段 schema 标准化**：§12.1 的 `capabilities` 与 `metrics` 内容由实现自定，待积累足够实现样本后由后续 CR 再标准化。
- **跨集群联邦查询**：在多个 Anchor Node 之间联合查询拓扑，属 NPS-AaaS Profile L3 / NPS Cloud 范畴，本节仅覆盖单 Anchor。
- **授权模型**：`NWP-TOPOLOGY-UNAUTHORIZED` 仅保留为 wire 信号；具体何时触发它的策略推迟到后续 CR（NeuronHub 商业部署诉求驱动）。
- **浏览器端传输（WebSocket）**：`npsd` 是否暴露 WebSocket 端点供浏览器消费另案跟踪。本节定义的拓扑查询语义与传输层无关。

---

## 13. 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NWP-AUTH-NID-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Agent scope 不覆盖目标节点 |
| `NWP-AUTH-NID-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | NID 证书已过期 |
| `NWP-AUTH-NID-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | NID 已被吊销 |
| `NWP-AUTH-NID-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | NID 颁发者不在 trusted_issuers 中 |
| `NWP-AUTH-NID-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Agent 缺少节点要求的能力 |
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` | Agent 的 `assurance_level` 低于节点 `min_assurance_level`（或 ActionSpec 的 per-action 覆盖）。响应 SHOULD 在 `hint` 字段附 CA 注册 URL。（NPS-RFC-0003）|
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` | 接收 Node 的 `reputation_policy`（Phase 2 NWM 字段 —— 见 [NPS-RFC-0004 §4.4](rfcs/NPS-RFC-0004-nid-reputation-log.cn.md)）命中了对发起方 `subject_nid` 的 `reject_on` 规则。响应 SHOULD 携带匹配的 `incident` + `severity` + 日志条目 `seq` 便于追溯。产生此错误的字段形态在 NWP v0.8（Phase 2）落地；错误码本身在 NWP v0.7（Phase 1）就保留，让 SDK 可以提前识别而无需额外 spec bump。（NPS-RFC-0004）|
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
| `NWP-TOPOLOGY-UNAUTHORIZED` | `NPS-AUTH-FORBIDDEN` | 调用方无权读取该 Anchor 的拓扑（§12）。授权策略由实现自定，详见 §12.4 |
| `NWP-TOPOLOGY-UNSUPPORTED-SCOPE` | `NPS-CLIENT-BAD-PARAM` | 该 Anchor 不实现请求的 `topology.scope` |
| `NWP-TOPOLOGY-DEPTH-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | 请求的 `topology.depth` 超过该 Anchor 上限 |
| `NWP-TOPOLOGY-FILTER-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | `topology.filter` 含未识别 key |

---

## 14. 安全考量

### 14.1 Scope 强制校验
节点 MUST 在每次请求时校验 Agent NID 的 scope。超出 scope 的请求 MUST 返回 `NWP-AUTH-NID-SCOPE-VIOLATION`，不得返回任何数据。

### 14.2 SSRF 防护
Complex Node 解析子节点引用时，MUST 维护允许的节点 URL 前缀白名单，禁止访问内网地址（RFC 1918）。

### 14.3 Token Budget 强制执行
超过预算时，节点 SHOULD 优先裁剪响应内容（字段精简 → 摘要 → 截断记录数）；无法裁剪时 MUST 返回 `NWP-BUDGET-EXCEEDED`，不得直接截断数据。详见 [token-budget.cn.md §4.3](token-budget.cn.md)。

### 14.4 频率限制
节点 SHOULD 对每个 Agent NID 实施频率限制。超限时返回 `NWP-RATE-LIMIT-EXCEEDED` 并附加 `X-NWP-Rate-Reset` 头。未认证请求 SHOULD 使用 IP 维度限制。

### 14.5 Filter 注入防护
- 字段名 MUST 仅含字母/数字/下划线/点，长度 ≤ 128 字符
- `$regex` MUST 经过 ReDoS 检测；Filter 嵌套深度 ≤ 8
- 节点 MUST 使用参数化查询，禁止字符串拼接

### 14.6 callback_url 防滥用
- ActionFrame `callback_url` MUST 为 `https://` 前缀
- 节点 SHOULD 对回调 URL 做 SSRF 检查（禁止内网地址）
- 节点 SHOULD 对 callback 推送失败做指数退避重试（最多 3 次），之后放弃并标记任务为 `COMPLETED` 而非无限重试

### 14.7 拓扑读取
实现 §12 的 Anchor Node MUST 把 `topology.snapshot` 与 `topology.stream` 视作鉴权接口。在授权模型尚未标准化前（§12.4），实现 MUST 至少在 L2 要求 `auth.required = true`，并 SHOULD 仅放开给 `node_kind` 含 `anchor` 或 Agent scope 显式带 `topology:read` 的 NID。未授权调用方 MUST 收到 `NWP-TOPOLOGY-UNAUTHORIZED`，不得静默返回空结果，避免对集群成员关系做 oracle 攻击。

---

## 15. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.8 | 2026-04-27 | 新增 §12 **保留查询类型**，引入 `topology.*` 命名空间，强制于 NPS-AaaS Profile L2：`topology.snapshot`（QueryFrame，`type="topology.snapshot"`）与 `topology.stream`（SubscribeFrame，`type="topology.stream"`）。QueryFrame §6.1 与 SubscribeFrame §8.1 各新增可选顶层字段 `type` 用于选入保留类型。DiffFrame §8.2 `event_type` 通过保留订阅类型扩展枚举 —— `topology.stream` 增加 `member_joined` / `member_left` / `member_updated` / `anchor_state` / `resync_required`。新增 4 条错误码：`NWP-TOPOLOGY-UNAUTHORIZED`、`NWP-TOPOLOGY-UNSUPPORTED-SCOPE`、`NWP-TOPOLOGY-DEPTH-UNSUPPORTED`、`NWP-TOPOLOGY-FILTER-UNSUPPORTED`（§13）。新增 §14.7 拓扑读取安全节。原 §12 错误码 / §13 安全考量 / §14 变更历史顺延为 §13 / §14 / §15 以容纳新章节。详见 [NPS-CR-0002](cr/NPS-CR-0002-anchor-topology-queries.md)。 |
| 0.7 | 2026-04-26 | **破坏性。** §2.1 节点类型：移除 `Gateway Node`；替换为 **Anchor Node**（集群控制平面 + NOP 路由 —— 继承既有角色）与 **Bridge Node**（NPS↔非-NPS 协议翻译 —— 全新）。NWM `node_type` 枚举更新；遗留 `"gateway"` MUST 拒绝。Anchor Node 详细语义（§2.1 内嵌）覆盖成员分派 + 可选注册表；Bridge Node 语义覆盖 HTTP/gRPC/MCP/A2A 目标适配器。Depends-On 升级为 NDP v0.5，引入 `node_kind` 数组形式 + `cluster_anchor` + `bridge_protocols` 字段。详见 [NPS-CR-0001](cr/NPS-CR-0001-anchor-bridge-split.md)。 |
| 0.6 | 2026-04-25 | NWM 新增可选顶层字段 `min_assurance_level`（`anonymous` / `attested` / `verified`），允许在单个 ActionSpec 上以 `auth.min_assurance_level` 做 per-action 覆盖。新增错误码 `NWP-AUTH-ASSURANCE-TOO-LOW`（`NPS-AUTH-FORBIDDEN`）。`Depends-On` 升级为 NCP v0.6（NPS-RFC-0001）+ NIP v0.4（NPS-RFC-0003）。详见 [NPS-RFC-0003](rfcs/NPS-RFC-0003-agent-identity-assurance-levels.cn.md)。 |
| 0.4 | 2026-04-14 | §3.2 新增 `/actions` 子路径；§4.1 NWM 新增 `actions` 字段；§4.2 capabilities 新增 stream_query、aggregate；§4.6 NWM Action 注册表（ActionSpec、params_anchor/result_anchor/async/idempotent）；QueryFrame §6.1 新增 `stream`、`aggregate`、`request_id`；§6.6 流式查询协议（StreamFrame 序列、estimated_total、提前终止）；§6.7 聚合查询（COUNT/SUM/AVG/MIN/MAX/COUNT_DISTINCT、group_by、having）；ActionFrame §7.1 新增 `request_id`；SubscribeFrame §8.1 新增 `resume_from_seq`；§8.2 DiffFrame 扩展字段（seq 单调递增、event_type、timestamp）及断线恢复语义；§9.1/9.2 新增 X-NWP-Request-ID；§9.4 HTTP 模式错误响应格式（application/nwp-error+json）；§10 更新完整示例（含错误响应）；§13.6 callback_url 防滥用安全节；新增 5 条错误码（AGGREGATE-UNSUPPORTED/-INVALID、STREAM-UNSUPPORTED、SUBSCRIBE-SEQ-TOO-OLD、task cancel 系列）|
| 0.3 | 2026-04-14 | SubscribeFrame (0x12)；auto_anchor；Filter $not/$exists/$regex；ActionFrame callback_url/priority；system.task.*；NWM min_agent_version/rate_limits；§14.4/14.5 安全节 |
| 0.2 | 2026-04-12 | 统一端口 17433；AnchorFrame 改为 Node 发布；NPT 计量；NPS 状态码映射 |
| 0.1 | 2026-04-10 | 初始规范 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
