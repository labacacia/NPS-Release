# NPS-1: Neural Communication Protocol (NCP)

**Spec Number**: NPS-1  
**Status**: Draft  
**Version**: 0.4  
**Date**: 2026-04-14  
**Port**: 17433（默认，全协议族共用）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  

> 本文档为 NCP 详细规范。套件总览见 [NPS-0-Overview.md](NPS-0-Overview.md)。

---

## 1. Terminology（术语）

本文档中的关键字 "MUST"、"MUST NOT"、"REQUIRED"、"SHOULD"、"SHOULD NOT"、"MAY" 按照 RFC 2119 解释。

---

## 2. 协议概述

NCP 是 NPS 的帧格式与通信基础层，定义 AI 节点间的帧结构、编码层级和语义压缩机制。所有上层协议（NWP/NIP/NDP/NOP）均以 NCP 帧为载体传输。

### 2.1 角色

- **Sender**：发起帧的一方（Agent 或节点）
- **Receiver**：接收并处理帧的一方
- **Relay**：透明转发帧，不修改 Payload（可选，用于代理场景）

### 2.2 传输模式

NCP 支持两种传输模式，帧格式完全一致，仅承载方式不同：

| 模式 | 承载 | 适用场景 | 推荐阶段 |
|------|------|---------|---------|
| **HTTP 模式** | NCP 帧序列化在 HTTP Body 中，配合 `X-NWP-*` 请求头 | Overlay 部署、与现有 Web 基础设施兼容、防火墙友好 | Phase 1 推荐 |
| **原生模式** | 直接 TCP/QUIC 连接，NCP 帧为线上格式，无 HTTP 开销 | 高性能 Agent-to-Agent、低延迟场景 | Phase 2+ |

**HTTP 模式**

```
POST /nwp/products/query HTTP/1.1
Host: api.example.com:17433
Content-Type: application/nwp-frame
X-NWP-Agent: urn:nps:agent:ca.innolotus.com:550e8400

[NCP Frame bytes / JSON]
```

**原生模式**

```
TCP connect → api.example.com:17433
→ HelloFrame (握手，0x06)
← CapsFrame (能力协商，0x04)
→ QueryFrame (0x10)
← CapsFrame (响应，0x04)
```

两种模式的帧 Payload 完全相同。原生模式下，错误通过 ErrorFrame (0xFE) 返回；HTTP 模式下，同时返回 HTTP 状态码（映射见 [status-codes.md](status-codes.md)）。

### 2.3 统一端口

NPS 全协议族默认共用 **端口 17433**。不同协议帧通过帧类型码（Frame Type）路由：

| 帧类型范围 | 协议 |
|-----------|------|
| 0x01–0x0F | NCP |
| 0x10–0x1F | NWP |
| 0x20–0x2F | NIP |
| 0x30–0x3F | NDP |
| 0x40–0x4F | NOP |
| 0xF0–0xFF | Reserved（含 ErrorFrame 0xFE）|

实现 MAY 为各协议分配独立端口用于隔离部署，但默认行为 MUST 支持单端口复用。

### 2.4 帧交换模式

| 模式 | 描述 | 典型帧序列 |
|------|------|-----------|
| 请求-响应 | 单次查询或操作 | QueryFrame → CapsFrame |
| 流式推送 | 大数据集或实时数据 | QueryFrame → StreamFrame × N → StreamFrame(is_last=true) |
| 增量订阅 | 变更通知 | 订阅请求 → DiffFrame × N |
| 多 Agent 同步 | 任务状态对齐 | AlignStream × N（见 NPS-5） |

### 2.5 连接状态机

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

### 2.6 连接握手序列（原生模式）

原生模式下，连接建立 MUST 遵循以下握手流程：

```
Client (Agent)                        Server (Node)
      │                                     │
      │── TCP/QUIC connect ──────────────→  │
      │── HelloFrame (0x06) ─────────────→  │  声明客户端版本与能力
      │                                     │  版本协商 & 能力交集计算
      │  ←──────────────── CapsFrame (0x04) │  返回协商后的服务端能力
      │                                     │  (若不兼容则返回 ErrorFrame + 断开)
      │  [可选] 预加载 AnchorFrame            │
      │── GET /.schema ──────────────────→  │
      │  ←──────────────── AnchorFrame(s)   │
      │                                     │
      │  ── ESTABLISHED ───────────────── ESTABLISHED
```

**版本协商规则**

- Server MUST 选择 min(client.nps_version, server.nps_version) 作为会话版本
- 若 client.min_version > server.nps_version，Server MUST 返回 `NCP-VERSION-INCOMPATIBLE` 并关闭连接
- 协商后的编码格式 = client.supported_encodings ∩ server.supported_encodings 的最优交集（优先 Tier-2）
- 协商后的 `max_frame_payload` = min(client.max_frame_payload, server.max_frame_payload)

**错误处理**

若握手失败（版本不兼容、能力集为空等），Server MUST 返回：

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

## 3. 帧格式

### 3.1 固定头（4 字节，默认） / 扩展头（8 字节，大帧模式）

**默认帧头（4 字节）**

```
 Byte 0          Byte 1          Byte 2–3
┌───────────────┬───────────────┬───────────────────────────┐
│  Frame Type   │     Flags     │      Payload Length       │
│   (1 byte)    │   (1 byte)    │        (2 bytes, BE)      │
└───────────────┴───────────────┴───────────────────────────┘
```

**扩展帧头（8 字节，EXT=1 时）**

```
 Byte 0          Byte 1          Byte 2–5               Byte 6–7
┌───────────────┬───────────────┬───────────────────────┬──────────────┐
│  Frame Type   │     Flags     │   Payload Length      │   Reserved   │
│   (1 byte)    │   (1 byte)    │   (4 bytes, BE)       │  (2 bytes)   │
└───────────────┴───────────────┴───────────────────────┘──────────────┘
```

- **Frame Type**：帧类型码，见帧命名空间（§2.3）
- **Flags**：控制标志位，见 §3.2
- **Payload Length**：默认 2 字节（最大 65,535），扩展模式 4 字节（最大 4,294,967,295）

### 3.2 Flags 字段（逐位定义）

```
Bit 7   Bit 6   Bit 5   Bit 4   Bit 3   Bit 2   Bit 1   Bit 0
┌───────┬───────┬───────┬───────┬───────┬───────┬───────┬───────┐
│  EXT  │  RSV  │  RSV  │  RSV  │  ENC  │ FINAL │  T1   │  T0  │
└───────┴───────┴───────┴───────┴───────┴───────┴───────┴───────┘
```

| 位 | 名称 | 描述 |
|----|------|------|
| 0–1 | T0, T1 | 编码 Tier：`00`=Tier-1 JSON，`01`=Tier-2 MsgPack，`10`=Reserved（原 Tier-3 位），`11`=Reserved |
| 2 | FINAL | 流式帧最终块标志（StreamFrame 专用，其他帧固定为 1） |
| 3 | ENC | Payload 使用应用层 E2E 加密（见 §7.4）；TLS 传输层加密独立，开发模式可为 0 |
| 4–6 | RSV | 保留，MUST 为 0，接收方 MUST 忽略 |
| 7 | EXT | 扩展帧头标志：`0`=默认 4 字节帧头（Payload ≤ 64KB），`1`=扩展 8 字节帧头（Payload ≤ 4GB） |

### 3.3 帧大小配置

| 配置项 | 默认值 | 最大值 | 说明 |
|--------|--------|--------|------|
| `max_frame_payload` | 65,535 (64KB) | 4,294,967,295 (4GB) | 单帧最大 Payload 字节数 |

- 默认帧头（EXT=0）：Payload 最大 **65,535 字节**，适用于绝大多数场景
- 扩展帧头（EXT=1）：Payload 最大 **4,294,967,295 字节**，适用于大 embedding 或文档批量传输
- 实现 MUST 支持默认模式，SHOULD 支持扩展模式
- Payload 超过当前连接协商的 `max_frame_payload` 时，发送方 MUST 使用 StreamFrame 分片传输
- `max_frame_payload` 的协商通过 CapsFrame 在连接建立时完成

**分片规则**

- StreamFrame 分片：seq 从 0 递增，最后一片 FINAL=1
- 分片内每帧 MUST 遵守当前连接的 `max_frame_payload` 限制

---

## 4. 帧类型

### 4.1 AnchorFrame (0x01)

Schema 锚点帧，由 **Node 发布**，用于建立全局 Schema 引用，消除后续重复传输。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x01` |
| `anchor_id` | string | 必填 | Schema 的 SHA-256 摘要，格式：`sha256:{64位hex}` |
| `schema` | object | 必填 | Schema 定义对象 |
| `schema.fields` | array | 必填 | 字段描述数组，见下 |
| `ttl` | uint32 | 可选 | 缓存有效期（秒），默认 3600，0 表示不缓存 |

**schema.fields 元素**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `name` | string | 必填 | 字段名 |
| `type` | string | 必填 | 数据类型：`string`/`uint64`/`int64`/`decimal`/`bool`/`timestamp`/`bytes`/`object`/`array` |
| `semantic` | string | 可选 | 语义标注，格式：`{domain}.{concept}`（见语义类型系统）|
| `nullable` | bool | 可选 | 是否可为 null，默认 false |

**anchor_id 计算规则**

1. 将 `schema` 字段按 **RFC 8785（JSON Canonicalization Scheme, JCS）** 序列化为规范化 JSON  
   — JCS 保证：所有 Unicode 字符规范化、对象 key 按 UTF-16 排序、无多余空白  
   — 实现 MUST 使用 JCS 标准库（不得自行实现），以确保跨语言 SDK（.NET/Python/TS）一致
2. 计算 JCS 输出的 UTF-8 字节流的 SHA-256 摘要
3. 格式化为 `sha256:{lowercase hex}`

> **注**：JCS 参考实现列表见 https://www.rfc-editor.org/rfc/rfc8785#appendix-A

**示例**

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

**语义类型系统（部分）**

| Semantic | 含义 |
|----------|------|
| `entity.id` | 实体唯一标识符 |
| `entity.label` | 实体人类可读名称 |
| `entity.description` | 实体描述文本 |
| `entity.timestamp.created` | 创建时间 |
| `entity.timestamp.updated` | 更新时间 |
| `commerce.price.usd` | 美元价格 |
| `commerce.price.cny` | 人民币价格 |
| `commerce.inventory.count` | 库存数量 |
| `geo.latitude` | 纬度 |
| `geo.longitude` | 经度 |

---

### 4.2 DiffFrame (0x02)

增量数据帧，仅传输变更字段的 patch，适用于订阅或轮询场景。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x02` |
| `anchor_ref` | string | 必填 | 基准 Schema 的 anchor_id |
| `base_seq` | uint64 | 必填 | 本 Diff 基于的版本序号 |
| `patch_format` | string | 可选 | 补丁格式：`json_patch`（默认）或 `binary_bitset`（Tier-2 推荐）|
| `patch` | array \| bytes | 必填 | 补丁数据（格式由 patch_format 决定，见下）|
| `entity_id` | string | 可选 | 被更新实体的 ID |

**patch_format: json_patch**（默认，Tier-1 JSON 场景）

`patch` 为 JSON Patch 操作数组（RFC 6902），每个元素含 `op`、`path`、`value`。

**patch_format: binary_bitset**（Tier-2 MsgPack 场景，减少 15–20% 体积）

`patch` 为二进制字节流（MsgPack `bin` 类型），结构如下：

```
┌──────────────────────────────────────────────────────┐
│  Changed Fields Bitset（ceil(N/8) bytes，N=Schema字段数）│
│  Bit i=1 表示第 i 个字段有变更（按 schema.fields 顺序）  │
├──────────────────────────────────────────────────────┤
│  Values（仅包含变更字段的新值，按字段顺序，MsgPack 编码）  │
└──────────────────────────────────────────────────────┘
```

规则：
- 发送方 MUST 仅在 Tier-2（MsgPack）编码帧中使用 `binary_bitset`
- 接收方若不支持 `binary_bitset`，MUST 返回 `NCP-DIFF-FORMAT-UNSUPPORTED`
- `patch_format` 省略时默认为 `json_patch`

**示例（json_patch）**

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

---

### 4.3 StreamFrame (0x03)

流式数据块帧，用于大数据集、实时推送或超帧大小的 Payload 分片。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x03` |
| `stream_id` | string | 必填 | 流的唯一标识符（UUID v4） |
| `seq` | uint32 | 必填 | 块序号，从 0 递增，接收方按 seq 重组 |
| `is_last` | bool | 必填 | true 表示此为最后一块 |
| `anchor_ref` | string | 可选 | Schema anchor_id（首块携带，后续块可省略） |
| `data` | array | 必填 | 本块数据，符合 anchor_ref 所指 Schema |
| `window_size` | uint32 | 可选 | 接收方可缓冲的剩余帧数（见流量控制语义）|
| `error_code` | string | 可选 | 非空时表示流异常终止，is_last 强制为 true |

**流量控制语义**

NCP 提供应用层语义背压，补充 TCP/QUIC 传输层流量控制：

- **初始窗口**：发送方在首帧（seq=0）携带 `window_size`，声明接收方剩余可接受帧数
- **窗口消耗**：每发送一帧，接收方逻辑窗口减 1
- **窗口更新**：接收方准备好接收更多数据时，发送一个反向 StreamFrame（`data=[]`, `is_last=false`），携带新的 `window_size` 以刷新窗口
- **背压暂停**：接收方发送 `window_size=0` 的更新帧，发送方 MUST 暂停，直到收到非零 `window_size`
- **省略语义**：`window_size` 未设置时，表示不启用应用层流量控制（依赖底层传输）
- 发送方在窗口耗尽后继续发送，接收方 SHOULD 返回 `NCP-STREAM-WINDOW-OVERFLOW` 并可终止流

---

### 4.4 CapsFrame (0x04)

封装完整响应体，最常见的响应帧类型。也用于连接建立时的能力协商。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x04` |
| `anchor_ref` | string | 必填 | Schema anchor_id |
| `count` | uint32 | 必填 | `data` 数组长度，MUST 等于 `len(data)` |
| `data` | array | 必填 | 数据记录数组，每条记录符合 anchor_ref Schema |
| `next_cursor` | string | 可选 | 下一页游标，Base64-URL 编码，null 表示最后一页 |
| `token_est` | uint32 | 可选 | 本响应预估 NPT 消耗（见 [token-budget.md](token-budget.md)）|
| `tokenizer_used` | string | 可选 | 实际使用的 tokenizer 标识 |
| `cached` | bool | 可选 | true 表示本响应来自服务端缓存 |
| `inline_anchor` | object | 可选 | Schema 已更新时内联返回最新 AnchorFrame，避免额外 RTT（见 §5.4）|

**连接协商 CapsFrame**

在原生模式连接建立时，Node 通过 CapsFrame 返回协商后的能力声明：

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

**数据响应示例**

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

> ⚠️ AlignFrame 已在 NCP v0.2 中标记为 **Deprecated**。  
> 请使用 NOP AlignStream (0x43)，见 [NPS-5-NOP.md](NPS-5-NOP.md)。  
> AlignFrame 将在 NPS v1.0 中移除。

---

### 4.6 HelloFrame (0x06)

原生模式客户端握手帧，**由 Agent（客户端）在连接建立后首先发送**，声明自身协议版本与能力。Server 以 CapsFrame 响应（见 §2.6）。

> **命名说明**：NIP 层已定义 IdentFrame (0x20) 用于身份证书交换。NCP HelloFrame 仅负责版本与能力协商，不携带身份信息，两者职责明确分离。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x06` |
| `nps_version` | string | 必填 | 客户端支持的最高 NPS 版本，格式 `"major.minor"` |
| `min_version` | string | 可选 | 客户端支持的最低 NPS 版本，默认与 `nps_version` 相同 |
| `supported_encodings` | array[string] | 必填 | 支持的编码格式列表，如 `["json", "msgpack"]`，按优先级降序 |
| `supported_protocols` | array[string] | 必填 | 支持的上层协议列表，如 `["ncp", "nwp", "nip"]` |
| `agent_id` | string | 可选 | Agent 的 NID（NIP 身份标识符），格式 `urn:nps:agent:{domain}:{id}` |
| `max_frame_payload` | uint32 | 可选 | 客户端可接受的最大帧 Payload 字节数，默认 65535 |
| `ext_support` | bool | 可选 | 是否支持扩展帧头（EXT=1），默认 false |
| `max_concurrent_streams` | uint32 | 可选 | 客户端可处理的最大并发流数，默认 32 |
| `e2e_enc_algorithms` | array[string] | 可选 | 客户端支持的 E2E 加密算法列表（见 §7.4），如 `["aes-256-gcm"]` |

**规则**

- HelloFrame MUST 是原生模式下客户端发送的第一个帧，发送前 MUST NOT 发送任何其他帧
- HTTP 模式不使用 HelloFrame（能力通过 `X-NWP-*` 请求头传递）
- HelloFrame 不携带加密（ENC=0），Tier 建议使用 Tier-1 JSON（握手阶段，编码尚未协商）
- Server 收到 HelloFrame 后，MUST 在 5 秒内回复 CapsFrame 或 ErrorFrame，否则客户端 SHOULD 断开连接

**HelloFrame 示例**

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

NPS 统一错误帧，所有协议层共用。在原生模式下承载错误响应。

**字段定义**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0xFE` |
| `status` | string | 必填 | NPS 状态码（见 [status-codes.md](status-codes.md)）|
| `error` | string | 必填 | 协议级错误码（如 `NCP-ANCHOR-NOT-FOUND`）|
| `message` | string | 可选 | 人类可读错误描述 |
| `details` | object | 可选 | 错误相关的结构化数据（如 `anchor_ref`、`stream_id`）|

**示例**

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

## 5. Schema 锚定机制

### 5.1 Schema 所有权

AnchorFrame 由 **Node（数据模型拥有者）** 发布。Agent 通过读取 Node 清单或 Schema 端点获取 AnchorFrame，本地缓存后在后续请求中引用。

### 5.2 标准流程

```
Agent                              Node
  │                                  │
  │── GET /.nwm ─────────────────→│  读取节点清单，获取 schema_anchors
  │←── NWM(schema_anchors) ───────│
  │                                  │
  │── GET /.schema ──────────────→│  获取完整 AnchorFrame（可选，按需）
  │←── AnchorFrame(schema) ───────│  Agent 本地缓存 anchor_id → schema
  │                                  │
  │── QueryFrame(anchor_ref) ────→│  请求只携带 anchor_ref
  │←── CapsFrame(anchor_ref) ─────│  响应也只引用 anchor_ref
  │                                  │
  │── QueryFrame(anchor_ref) ────→│  后续请求无需重传 Schema
  │←── CapsFrame(anchor_ref) ─────│
```

### 5.3 缓存语义

- Agent MUST 缓存从 Node 获取的 AnchorFrame，以 `anchor_id` 为 key
- 缓存有效期为 AnchorFrame 中的 `ttl` 秒
- Agent SHOULD 在连接建立后预加载所有可能用到的 AnchorFrame

### 5.4 缓存未命中处理

#### 5.4.1 Schema 已更新（过期 anchor_ref）

若 Agent 引用了已过期的 `anchor_ref`，Node 发现该 Schema 版本已更新时，处理行为取决于请求端 `auto_anchor` 标志（在 NWP QueryFrame 中定义）：

- **auto_anchor=true（默认）**：Node SHOULD 在 CapsFrame 响应中附加 `inline_anchor` 字段，包含最新 AnchorFrame，避免额外 RTT：

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

  Agent 收到含 `inline_anchor` 的响应后，MUST 更新本地缓存（替换旧 `anchor_id` 对应的 Schema）。

- **auto_anchor=false**：Node SHOULD 返回带旧 anchor_id 的 `NCP-ANCHOR-STALE`，由 Agent 主动请求新 AnchorFrame。

#### 5.4.2 未知 anchor_ref

若 Agent 引用了 Node 完全不认识的 `anchor_ref`（如手工构造、跨节点误引用），Node MUST 返回：

```json
{
  "frame": "0xFE",
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NCP-ANCHOR-NOT-FOUND",
  "details": { "anchor_ref": "sha256:..." }
}
```

> **注**：`NCP-ANCHOR-STALE` 表示 anchor 存在但已更新；`NCP-ANCHOR-NOT-FOUND` 表示 anchor 从未被此 Node 注册。

---

## 6. 状态码与错误码

NPS 采用两级错误体系：

1. **NPS 状态码**：传输层状态分类，见 [status-codes.md](status-codes.md)
2. **协议错误码**：具体错误标识，前缀标识所属协议

### NCP 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NCP-ANCHOR-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | anchor_ref 引用的 Schema 不存在 |
| `NCP-ANCHOR-SCHEMA-INVALID` | `NPS-CLIENT-BAD-FRAME` | AnchorFrame 中 Schema 格式不合法 |
| `NCP-ANCHOR-ID-MISMATCH` | `NPS-CLIENT-CONFLICT` | 相同 anchor_id 收到不同 Schema（锚点污染攻击防御）|
| `NCP-FRAME-UNKNOWN-TYPE` | `NPS-CLIENT-BAD-FRAME` | 未知帧类型码 |
| `NCP-FRAME-PAYLOAD-TOO-LARGE` | `NPS-LIMIT-PAYLOAD` | Payload 超过协商的 max_frame_payload |
| `NCP-FRAME-FLAGS-INVALID` | `NPS-CLIENT-BAD-FRAME` | Flags 字段中保留位非零 |
| `NCP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | StreamFrame 序号不连续 |
| `NCP-STREAM-NOT-FOUND` | `NPS-STREAM-NOT-FOUND` | stream_id 引用的流不存在 |
| `NCP-STREAM-LIMIT-EXCEEDED` | `NPS-STREAM-LIMIT` | 超出单连接最大并发流数 |
| `NCP-ENCODING-UNSUPPORTED` | `NPS-SERVER-ENCODING-UNSUPPORTED` | 请求的编码 Tier 不被支持 |
| `NCP-ANCHOR-STALE` | `NPS-CLIENT-CONFLICT` | anchor_ref 存在但 Schema 已更新（配合 inline_anchor 使用）|
| `NCP-DIFF-FORMAT-UNSUPPORTED` | `NPS-CLIENT-BAD-FRAME` | DiffFrame 使用了接收方不支持的 patch_format |
| `NCP-VERSION-INCOMPATIBLE` | `NPS-PROTO-VERSION-INCOMPATIBLE` | 客户端 min_version 高于 Server 支持版本 |
| `NCP-STREAM-WINDOW-OVERFLOW` | `NPS-STREAM-LIMIT` | 发送方在窗口耗尽后继续发送 |

HTTP 模式下的状态码映射见 [status-codes.md](status-codes.md)。

---

## 7. 安全考量

### 7.1 重放攻击防御
CapsFrame 和 QueryFrame 应在 TLS 层由传输协议防御重放。若在非 TLS 场景（开发模式）下，Agent SHOULD 在请求中携带 `nonce` 字段。

### 7.2 锚点污染（Anchor Poisoning）
Schema 由 Node 发布，Agent 只读引用。Node MUST 对 anchor_id 做幂等校验——相同 anchor_id 对应的 Schema MUST 始终一致。Agent 若检测到同一 anchor_id 返回了不同 Schema，SHOULD 断开连接并报告安全事件。

### 7.3 流式洪泛（Stream Flooding）
Node SHOULD 限制单连接最大并发流数（推荐默认值：32，通过 CapsFrame `max_concurrent_streams` 协商）。超出限制时返回 `NCP-STREAM-LIMIT-EXCEEDED`。

### 7.4 E2E 加密（ENC 标志正式定义）

当帧头 Flags **ENC=1** 时，帧 Payload 使用应用层端到端加密。E2E 加密独立于 TLS 传输层，适用于多跳 Relay 场景中 Relay 不可信的情况。

**算法选择**

通过握手 CapsFrame 的 `e2e_enc_algorithms` 字段协商，优先级从高到低：

| 算法 | 标识符 | 密钥长度 | Nonce | Tag |
|------|--------|----------|-------|-----|
| AES-256-GCM | `aes-256-gcm` | 256-bit | 12 bytes | 16 bytes |
| ChaCha20-Poly1305 | `chacha20-poly1305` | 256-bit | 12 bytes | 16 bytes |

**ENC=1 时 Payload 布局**

```
┌────────────────┬──────────────────────────────────────┬─────────────┐
│   Nonce        │   Encrypted Payload                  │  Auth Tag   │
│  (12 bytes)    │   (原始 Payload 密文)                 │  (16 bytes) │
└────────────────┴──────────────────────────────────────┴─────────────┘
```

- `Payload Length`（帧头中）= 12 + len(encrypted) + 16
- Nonce MUST 对每个帧唯一，推荐使用随机生成（CSPRNG）
- AAD（Additional Authenticated Data）= Frame Header（4 或 8 字节）
- 密钥管理：E2E 密钥通过 NIP（Neural Identity Protocol）的密钥交换机制分发，不在 NCP 层定义

**实现要求**

- 若 HelloFrame 中 `e2e_enc_algorithms` 为空，表示不支持 E2E 加密，两端 MUST NOT 设置 ENC=1
- 若 ENC=1 但会话未协商 E2E 算法，接收方 MUST 返回 `NCP-ENC-NOT-NEGOTIATED` 并丢弃该帧
- 开发/调试模式下（仅限非生产环境），ENC 可为 0，但实现 SHOULD 在日志中警告

**新增错误码**

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NCP-ENC-NOT-NEGOTIATED` | `NPS-CLIENT-BAD-FRAME` | ENC=1 但会话未协商 E2E 加密算法 |
| `NCP-ENC-AUTH-FAILED` | `NPS-CLIENT-BAD-FRAME` | E2E 加密 Auth Tag 验证失败（可能被篡改）|

---

## 8. 编码层级

所有帧类型支持以下编码层级，通过帧头 Flags T0/T1 位标识：

| Tier | 格式 | Flag | 适用场景 |
|------|------|------|---------|
| Tier-1 | JSON | `00` | 开发调试、兼容模式 |
| Tier-2 | MsgPack（Binary） | `01` | 生产环境，~60% 体积压缩 |
| — | Reserved | `10` | 保留，供未来高性能编码格式使用 |
| — | Reserved | `11` | 保留 |

默认：生产环境使用 Tier-2，开发环境使用 Tier-1。

> **注**：原 Tier-3 MatrixTensor 概念已从编码层级中移除，`10` 位标记为 Reserved。若未来定义高性能编码格式（如向量/张量专用编码），将通过正式 RFC 流程分配。

---

## 9. 实现注意事项

- AnchorFrame 缓存上限建议 1000 条/连接，超出时 LRU 淘汰
- MsgPack（Tier-2）编解码推荐使用 MessagePack 官方库，不建议自行实现
- StreamFrame seq 溢出（uint32 到达 0xFFFFFFFF）时，新流 MUST 使用新 stream_id
- 扩展帧头模式（EXT=1）的支持 SHOULD 在 CapsFrame 能力协商中声明
- **anchor_id 计算**：必须使用 RFC 8785 JCS 标准库，禁止自行实现规范化逻辑（跨语言 SDK 一致性）
- **HelloFrame 超时**：Server 若 5 秒内未收到 HelloFrame，SHOULD 发送 ErrorFrame 并关闭连接
- **DiffFrame binary_bitset**：仅在双端通过 CapsFrame 协商支持后使用；Tier-1 JSON 模式下禁止使用
- **E2E 加密 Nonce**：每帧独立随机 Nonce（12 bytes, CSPRNG），禁止重用；密钥轮换频率 SHOULD 不低于每 2^32 帧一次
- **inline_anchor 处理**：Agent 收到 `inline_anchor` 后必须先验证 anchor_id（JCS+SHA-256），验证通过后再更新缓存
- **Tier-2 性能优化（.NET）**：`Tier2MsgPackCodec` 建议使用 MessagePack AOT 代码生成，避免高频帧解析的反射开销（实现建议，不影响协议互操作性）

---

## 10. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.4 | 2026-04-14 | 新增 IdentFrame (0x06) 握手帧；新增 §2.6 连接握手序列及版本协商规则；anchor_id 计算明确引用 RFC 8785 JCS；DiffFrame 新增 patch_format 字段（json_patch / binary_bitset）；CapsFrame 新增 inline_anchor；StreamFrame 流量控制语义正式化（window_size 协议）；§7.4 E2E 加密节（ENC 标志、AES-256-GCM / ChaCha20-Poly1305、Payload 布局）；§5.4 auto-anchor 协议（NCP-ANCHOR-STALE + inline_anchor）；新增错误码 NCP-ANCHOR-STALE、NCP-DIFF-FORMAT-UNSUPPORTED、NCP-VERSION-INCOMPATIBLE、NCP-STREAM-WINDOW-OVERFLOW、NCP-ENC-NOT-NEGOTIATED、NCP-ENC-AUTH-FAILED |
| 0.3 | 2026-04-12 | 传输双模（HTTP/原生）；统一端口 17433；可配置帧大小（EXT 位）；ErrorFrame (0xFE)；NPS 状态码体系；Tier-3 标记 Reserved；AnchorFrame 所有权明确为 Node 发布；Token 估算改用 NPT |
| 0.2 | 2026-04-10 | AnchorFrame/DiffFrame/StreamFrame/CapsFrame/AlignFrame 定义；Flags 字段逐位定义；AlignFrame 标记 Deprecated |
| 0.1 | 2026-03-01 | 初始帧格式与编码层级定义 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
