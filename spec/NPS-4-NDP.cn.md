[English Version](./NPS-4-NDP.md) | 中文版

# NPS-4: Neural Discovery Protocol (NDP)

**Spec Number**: NPS-4
**Status**: Proposed
**Version**: 0.7
**Date**: 2026-05-10
**Port**: 17433（默认，共用）/ 17436（可选独立）
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.6)、NPS-3 (NIP v0.8)

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
| `activation_mode` | string | 条件必填 | 取值 `ephemeral` / `resident` / `hybrid` 之一。声明符合 NPS-Node Profile L1+ 的发布者 MUST 携带；未合规发布者可不带。接收方 MUST 把缺失字段按 `ephemeral` 处理（与 NPS v1.0-alpha.2 发布者保持向后兼容）。见 §3.1.1。 |
| `activation_endpoint` | object | 条件必填 | `resident` / `hybrid` 模式的推送目标，结构同 `addresses[]` 每项。`activation_mode ∈ {resident, hybrid}` 时 MUST 存在；其他模式下 MUST 不出现。 |
| `spawn_spec_ref` | string | 可选 | 发布方 daemon 用于按需具现化 Agent 进程的不透明引用。对 `ephemeral` 以及 `hybrid` 冷启动有意义。内容 schema 在 NPS-Node Profile L3（未来 NPS-Daemon-Spec）中标准化。 |
| `node_roles` | string 数组 | 可选 | 该发布者承担的节点功能角色列表。每个值取自 `"memory"`、`"action"`、`"complex"`、`"anchor"`、`"bridge"`。遗留值 `"gateway"` 在 v1.0-alpha.3 移除（NPS-CR-0001），解析器 MUST 以 `NDP-ANNOUNCE-ROLE-REMOVED` 拒绝；其他无法识别的值 MUST 以 `NDP-ANNOUNCE-ROLE-UNKNOWN` 拒绝。单角色节点发送一元素数组（`"node_roles": ["memory"]`）。缺省表示"按 `node_type` 字段单角色"，接收方 SHOULD 回退到 `node_type`。解析器 MUST 在 alpha 过渡期内把遗留字段名 `node_kind` 当作 `node_roles` 的别名接受。（NPS-CR-0001；在 NDP v0.6 由 `node_kind` 更名而来——见 M1 命名消歧修复）|
| `cluster_anchor` | string (NID) | 可选 | 加入集群的非-Anchor 节点，标识它注册到的 Anchor Node 的 NID。独立节点和 Anchor Node 自己 MUST 不带。（NPS-CR-0001）|
| `bridge_protocols` | string 数组 | 可选 | 在 `node_roles` 中声明 `"bridge"` 的节点列出支持的外部协议。标准值 `"http"` / `"grpc"` / `"mcp"` / `"a2a"`。开放枚举；第三方适配器 MAY 通过未来 CR 注册更多值。未声明 `"bridge"` 的节点 MUST 不带。（NPS-CR-0001）|
| `signature` | string | 必填 | IdentFrame 私钥签名，防止伪造 |

**示例（L1+ 发布者，ephemeral）**

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
  "activation_mode": "ephemeral",
  "signature": "ed25519:..."
}
```

**示例（L2+ 发布者，resident）**

```json
{
  "frame": "0x30",
  "nid": "urn:nps:agent:labacacia:writer-42",
  "node_type": "agent",
  "addresses": [
    { "host": "10.0.0.5", "port": 17434, "protocol": "nwp" }
  ],
  "capabilities": ["nwp:invoke"],
  "ttl": 300,
  "timestamp": "2026-04-24T00:00:00Z",
  "activation_mode": "resident",
  "activation_endpoint": { "host": "10.0.0.5", "port": 17440, "protocol": "nwp" },
  "signature": "ed25519:..."
}
```

#### 3.1.1 激活语义

`activation_mode` 告诉接收方发布者期望哪种方向的通信：推送（push）还是拉取（pull）：

| 模式 | 发送方行为 | 接收方预期 |
|------|----------|----------|
| `ephemeral` | 走发布方的 per-NID inbox + pull 交付；帧到达时发布方进程可能不在运行。 | 帧进入发布方 inbox，等待发布方拉取。 |
| `resident` | 通过长连接向 `activation_endpoint` 推送。 | 发布方在声明的端点上接收推送帧。 |
| `hybrid` | 先尝试推送到 `activation_endpoint`；若在发布方 wake budget 内不可达，则回退到 inbox。 | 发布方收到第一帧时从 hibernate 中唤醒；唤醒后推送目标可用。 |

**向后兼容**：NPS v1.0-alpha.2 发布者不会发出 `activation_mode`。接收方 MUST 把缺失字段按 `ephemeral` 处理，确保 alpha.2 AnnounceFrame 仍能正确解析。接收方 MUST NOT 因为缺少该字段而拒绝一个 AnnounceFrame。

**合规参考**：合规框架及每一级的字段要求，见 [NPS-Node Profile §1.3 / §6](./services/NPS-Node-Profile.cn.md)。

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

> **版本字符串格式说明**：DNS TXT `v` 键的值使用紧凑格式 `nps1`，遵循 DNS TXT 记录惯例（小写、无空格）。NCP 原生模式连接 preamble 使用 RFC 风格令牌 `NPS/1.0\n`（见 NPS-1 §2.6.1）。二者均标识 NPS 协议第 1 版，但编码形式不同，作用于不同的协议层，不可互换。

---

## 6. 错误码

| 错误码 | NPS 状态码 | 描述 |
|--------|-----------|------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | nwp:// 地址无法解析 |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | 解析结果存在冲突（多个不一致的注册）|
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | 解析请求超时 |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame 签名验证失败 |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame 中 NID 与签名证书不一致 |
| `NDP-ANNOUNCE-ROLE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | `node_roles` 包含已移除的遗留值 `"gateway"`（NPS-CR-0001）；响应 SHOULD 携带指向 NPS-CR-0001 的 `hint` 字段 |
| `NDP-ANNOUNCE-ROLE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | `node_roles` 包含无法识别的值（`"gateway"` 遗留情况请用 `NDP-ANNOUNCE-ROLE-REMOVED`）|
| `NDP-ANNOUNCE-CONFLICT` | `NPS-CLIENT-CONFLICT` | 两个 AnnounceFrame 拥有相同的 `nid` 与 `graph_seq` 但内容不同（注册表投毒企图）|
| `NDP-GRAPH-SEQ-ROLLBACK` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame 的 `graph_seq` 小于或等于该 NID 已接受的最高值（回滚企图）|
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame 序号不连续 |
| `NDP-ISSUER-NOT-ALLOWED` | `NPS-AUTH-FORBIDDEN` | AnnounceFrame 的签发者（签名 CA）不在当前注册表 profile 的签发者白名单中 |
| `NDP-CA-ATTEST-REQUIRED` | `NPS-AUTH-UNAUTHENTICATED` | 当前注册表 profile 要求 CA 背书的 NID，但 AnnounceFrame 证书链未锚定到配置的信任根 |
| `NDP-REGISTRY-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | NDP Registry 暂时不可用 |

HTTP 模式下的状态码映射见 [status-codes.cn.md](status-codes.cn.md)。

---

## 7. 安全考量

### 7.1 Announce 防伪
接收方 MUST 验证 AnnounceFrame 中的 signature，确认发布者持有对应 NID 的私钥，防止伪造节点广播。

### 7.2 Registry 防污染
中心化 Registry（NPS Cloud）MUST 要求注册方提供有效 IdentFrame，并通过 NIP CA 验证后才接受 AnnounceFrame 入库。

### 7.3 注册表安全 Profile

每个 NDP Registry 部署 MUST 显式声明三种安全 profile 之一。Profile 是 Registry 自身的配置，而不是 AnnounceFrame 的配置：profile 决定 Registry 接受、保留、对外提供哪些 AnnounceFrame。实现 MUST 在未声明 profile 时拒绝启动（不存在隐式默认）。

| Profile | 签发者白名单 | CA 背书 NID | 重放窗口 | 联邦 |
|---------|-------------|------------|---------|------|
| `local-dev` | 不强制 | 不要求 | `0`（重放防御关闭）| 不允许 |
| `org-private` | 必填（CA 证书指纹集合）| SHOULD | `300s` | 不允许 |
| `public-federated` | 通过 CA 信任链强制 | MUST | `300s` | 允许（需双边信任协议，见 §7.6）|

**`local-dev`**：仅用于单机或单开发者环境。如果 Registry 监听任何非 loopback 接口，MUST 拒绝以 `local-dev` 启动，除非操作员设置覆盖标志；该覆盖 MUST 在每次启动日志中记录。任何承载真实流量的服务 MUST NOT 使用 `local-dev`。

**`org-private`**：单一组织的内网注册表。运营方配置非空的 CA 证书指纹集合作为签发者白名单；签名链未终结于白名单内 CA 的 AnnounceFrame MUST 以 `NDP-ISSUER-NOT-ALLOWED` 拒绝。CA 背书的 NID（其所有权由组织 CA 背书的 NID，见 NPS-3 §6）SHOULD 强制；若强制且缺失，以 `NDP-CA-ATTEST-REQUIRED` 拒绝。跨注册表联邦 MUST 关闭。

**`public-federated`**：公网注册表（如 NPS Cloud Registry）。CA 背书 NID MUST 强制：证书链未锚定到已配置的公开信任根的 AnnounceFrame，Registry MUST 以 `NDP-CA-ATTEST-REQUIRED` 拒绝。签发者白名单隐含在该信任根集合中。跨注册表联邦 MAY 按 §7.6 启用。

### 7.4 AnnounceFrame 防投毒

无论何种 profile，每个 NDP Registry MUST 强制以下规则：

1. **签名覆盖范围**。`signature` 字段 MUST 至少覆盖 `(nid, graph_seq, timestamp)` 三元组以及 AnnounceFrame body 其余部分。接收方 MUST 在任何注册表侧状态变更前完成签名验证；签名验证 MUST 先于去重、冲突检测和持久化。
2. **重放防御**。`timestamp` 偏离当前时间超过 profile 重放窗口（过去或未来）的 AnnounceFrame，MUST 以 `NDP-ANNOUNCE-SIGNATURE-INVALID` 拒绝。重放窗口 `0`（仅 `local-dev` profile）禁用该检查。
3. **重复抑制**。`(nid, graph_seq)` 与已接受条目完全相同的 AnnounceFrame，MUST 静默丢弃（不返回错误，不变更状态）。多播环境下的字节级重复属正常现象。
4. **冲突拒绝**。`(nid, graph_seq)` 与已有条目相同但所覆盖内容不同的 AnnounceFrame，MUST 以 `NDP-ANNOUNCE-CONFLICT` 拒绝。被拒帧与已有条目 SHOULD 同步记入审计日志供运营方排查；冲突帧是私钥泄露或发布者集群配置错误的证据。

### 7.5 Graph 序列回滚防御

AnnounceFrame（以及对应 GraphFrame）的 `graph_seq` 字段是 per-NID 单调递增计数器。接收方 MUST 跟踪每个 NID 已接受的最高 `graph_seq`。`graph_seq` 小于或等于该 NID 已跟踪最高值的 AnnounceFrame，MUST 以 `NDP-GRAPH-SEQ-ROLLBACK` 拒绝。这防止攻击者捕获旧 AnnounceFrame 后再次发布以使节点状态降级。在 `org-private` 与 `public-federated` profile 下，已跟踪最高值 MUST 在 Registry 重启后持久化；`local-dev` MAY 仅保存在内存中。

### 7.6 跨注册表联邦

联邦——从对等注册表导入 AnnounceFrame——仅在 `public-federated` profile 下允许。两个 Registry 之间的联邦需要 **双边信任协议**：

1. 双方运营方 MUST 通过带外渠道交换 CA 根证书集合；各方将对方的根证书配置为联邦专用信任集合，与本机原生信任集合区分。
2. 从联邦对等方导入的 AnnounceFrame，MUST 用导入侧本地 CA 信任链（原生 ∪ 联邦专用）重新校验。在对等方有效的 AnnounceFrame 在本侧并不隐式有效。
3. 联邦通道本身 MUST 经过认证（基于 NIP 颁发证书的双向 TLS）。
4. 联邦条目 MUST 标注来源注册表标识，使下游消费者可基于来源应用差异化策略（例如排序、过滤）。
5. 任何一方 MAY 单方面撤销协议；撤销后导入侧 Registry MUST 在一个 TTL 窗口内使所有来自被撤销对等方的联邦条目失效。

联邦永不可传递：信任协议 A↔B 与 B↔C 不构成 A↔C。

### 7.7 注册表运营方信任级别

Registry 所采用的 profile 隐含相应的运营方信任级别。合规部署 MUST 满足其所声明 profile 的运营方信任级别要求。

| Profile | 运营方 | CA 范围 | 审计要求 |
|---------|-------|---------|---------|
| `local-dev` | 个人（单开发者 / CI 临时实例）| 无 / 自签 | 无 |
| `org-private` | 具名组织 | 组织 CA | 内部审计（年度签发者白名单复核、配置变更签名）|
| `public-federated` | 具备问责能力的法人主体（如 NPS Cloud）| 公开 CA 链 | 外部审计（独立第三方年度审计，覆盖签发者白名单溯源、联邦协议、密钥托管、事件响应）|

Registry MUST NOT 声明其无法满足相应运营方信任要求的 profile。

---

## 8. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.7 | 2026-05-10 | 新增 §7.3–§7.7：三个具名注册表安全 profile（`local-dev` / `org-private` / `public-federated`）、AnnounceFrame 防投毒规则（签名覆盖、重放防御、重复抑制、冲突拒绝）、graph_seq 回滚防御、跨注册表联邦要求、运营方信任级别表。新增错误码：`NDP-ANNOUNCE-CONFLICT`、`NDP-GRAPH-SEQ-ROLLBACK`、`NDP-ISSUER-NOT-ALLOWED`、`NDP-CA-ATTEST-REQUIRED`。（Issue #33）|
| 0.6 | 2026-05-01 | **破坏性更名（pre-1.0）**：`node_kind` 更名为 `node_roles`（仅接受 string 数组；废弃单字符串形式）。解析器 MUST 在 alpha.5 之前把 `node_kind` 当别名接受以保持向后兼容。新增约束：NWP NWM `node_type` MUST 为 `node_roles` 中的一项（见 NWP §2.1 节点角色解析）。修复 M1 命名消歧问题——`node_kind`（多角色发现字段）与 `node_type`（单运营角色）名字相近但语义分叉，且无跨协议约束文档。|
| 0.5 | 2026-04-26 | AnnounceFrame (0x30) 新增三个 additive 字段以支撑 NPS-CR-0001 —— `node_kind`（string 或 string 数组；取值 `"memory"` / `"action"` / `"complex"` / `"anchor"` / `"bridge"`；遗留 `"gateway"` 拒绝）、`cluster_anchor`（NID —— 标识非-Anchor 集群成员注册到的 Anchor）、`bridge_protocols`（string 数组 —— 给 `"bridge"` 节点；标准值 `"http"` / `"grpc"` / `"mcp"` / `"a2a"`）。全部 additive 且向后兼容：alpha.3 之前的发布者不发 `node_kind`，接收方回退到 `node_type`。Depends-On 升级为 NCP v0.6（NPS-RFC-0001）+ NIP v0.4（NPS-RFC-0003）。|
| 0.4 | 2026-04-24 | AnnounceFrame (0x30) 新增三个 additive 字段 —— `activation_mode`（NPS-Node Profile L1+ 必填）、`activation_endpoint`（`resident` / `hybrid` 必填）、`spawn_spec_ref`（L3，可选）。新增 §3.1.1 激活语义小节与 alpha.3 前发布者的向后兼容规则。`Depends-On` NCP 版本订正为 v0.5。 |
| 0.3 | 2026-04-19 | Status Draft → Proposed；双语统一（EN 主 + CN 副镜像）；无 wire 层变更 |
| 0.2 | 2026-04-12 | 统一端口 17433；错误码改用 NPS 状态码映射；完善错误码列表 |
| 0.1 | 2026-04-10 | 初始骨架：AnnounceFrame/ResolveFrame/GraphFrame、DNS TXT 规范、初始化流程 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
