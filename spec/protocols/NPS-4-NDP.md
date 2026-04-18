# NPS-4: Neural Discovery Protocol (NDP)

**Spec Number**: NPS-4  
**Status**: Draft  
**Version**: 0.2  
**Date**: 2026-04-12  
**Port**: 17433（默认，共用）/ 17436（可选独立）  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Depends-On**: NPS-1 (NCP v0.3), NPS-3 (NIP v0.2)  

---

## 1. Terminology

本文档中的关键字 "MUST"、"SHOULD"、"MAY" 按照 RFC 2119 解释。

---

## 2. 协议概述

NDP 是 AI 版 DNS。Agent 通过 NDP 解析 `nwp://` 地址到物理端点，广播自身能力，订阅节点图谱变更，支持去中心化和中心化两种网络拓扑。

### 2.1 解析模式

| 模式 | 适用场景 | 优先级 |
|------|---------|--------|
| 本地注册表（UDP Multicast）| 内网/局域网 | 最高 |
| DNS TXT 记录 | 公网，分布式 | 中 |
| NPS Cloud Registry | 全局，中心化 | 最低（兜底）|

---

## 3. 帧类型

### 3.1 AnnounceFrame (0x30)

节点或 Agent 广播自身存在与能力。

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x30` |
| `nid` | string | 必填 | 发布者 NID |
| `node_type` | string | 条件必填 | 节点类型（node 实体必填）|
| `addresses` | array | 必填 | 物理地址列表，每项含 `host`/`port`/`protocol` |
| `capabilities` | array | 必填 | 能力列表（复用 NIP capabilities 定义）|
| `ttl` | uint32 | 必填 | 广播有效期（秒），0 = 下线通知 |
| `timestamp` | string | 必填 | 广播时间（ISO 8601 UTC）|
| `signature` | string | 必填 | IdentFrame 私钥签名，防止伪造 |

**示例**

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

解析 `nwp://` URL 到物理端点。

**请求**

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x31` |
| `target` | string | 必填 | 待解析的 `nwp://` URL |
| `requester_nid` | string | 可选 | 请求方 NID（用于授权检查）|

**响应**

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

节点图谱同步与变更订阅。

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `frame` | uint8 | 必填 | 固定值 `0x32` |
| `initial_sync` | bool | 必填 | true = 全量推送，false = 增量 patch |
| `nodes` | array | 条件必填 | 全量同步时的节点列表 |
| `patch` | array | 条件必填 | 增量同步时的 JSON Patch（RFC 6902）|
| `seq` | uint64 | 必填 | 图谱版本序号，单调递增 |

---

## 4. Agent 初始化流程

```
Agent 启动
  │
  ├─ 1. NIP：加载 IdentFrame（从文件或 CA 自动获取）
  │
  ├─ 2. NDP：发送 AnnounceFrame（广播 Agent 上线）
  │
  ├─ 3. NDP：发送 GraphFrame 订阅请求（接收节点图谱）
  │         ← GraphFrame(initial_sync=true) 全量图谱
  │         ← GraphFrame(initial_sync=false) 增量更新（持续）
  │
  └─ 就绪：可发起 NWP 访问
```

---

## 5. DNS TXT 记录规范

节点可通过 DNS TXT 记录发布发现信息，无需中心化 Registry：

```
# 节点发现记录
_nps-node.api.example.com.  IN TXT  "v=nps1 type=memory port=17434 nid=urn:nps:node:api.example.com:products fp=sha256:a3f9..."

# CA 发现记录
_nps-ca.mycompany.com.      IN TXT  "v=nps1 ca=https://ca.mycompany.com/.well-known/nps-ca"
```

**TXT 记录字段**

| 键 | 必填 | 描述 |
|----|------|------|
| `v` | 必填 | 版本，固定 `nps1` |
| `type` | 可选 | 节点类型：`memory`/`action`/`complex` |
| `port` | 可选 | 端口（默认 17434）|
| `nid` | 必填 | 节点 NID |
| `fp` | 可选 | 节点证书指纹 |
| `ca` | 条件必填 | CA 发现端点（CA 记录必填）|

---

## 6. 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | nwp:// 地址无法解析 |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | 解析结果存在冲突（多个不一致的注册）|
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 解析请求超时 |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame 签名验证失败 |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame 中 NID 与签名证书不一致 |
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame 序号不连续 |
| `NDP-REGISTRY-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | NDP Registry 暂时不可用 |

HTTP 模式下的状态码映射见 [status-codes.md](status-codes.md)。

---

## 7. 安全考量

### 7.1 Announce 防伪
接收方 MUST 验证 AnnounceFrame 中的 signature，确认发布者持有对应 NID 的私钥，防止伪造节点广播。

### 7.2 Registry 防污染
中心化 Registry（NPS Cloud）MUST 要求注册方提供有效 IdentFrame，并通过 NIP CA 验证后才接受 AnnounceFrame 入库。

---

## 8. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.2 | 2026-04-12 | 统一端口 17433；错误码改用 NPS 状态码映射；完善错误码列表 |
| 0.1 | 2026-04-10 | 初始骨架：AnnounceFrame/ResolveFrame/GraphFrame、DNS TXT 规范、初始化流程 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
