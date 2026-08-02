[English Version](./NPS-Node-L2.md) | 中文版

# NPS-Node-L2 合规测试套件

**Status**: Draft
**Version**: 0.5
**Date**: 2026-08-01
**Applies-To**: [NPS-AaaS-Profile §4.3](../NPS-AaaS-Profile.cn.md) — Level 2 Standard
**Authors**: Ori Lynn / INNO LOTUS PTY LTD

> 本文档定义 Anchor Node 实现声明 NPS-AaaS-Profile L2 合规所 MUST 通过的测试用例，
> 当前版本仅覆盖 [NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md) 引入的
> **L2-08 拓扑读取** 要求。
>
> 其余 L2 要求（L2-01 至 L2-07 —— NOP 编排、OpenTelemetry 追踪、CGN Token Budget、
> 预检、重试 / 超时、异步 Action、AlignStream 背压）的测试用例由后续 CR 跟进，
> **不在本文档范围**。后续 CR 将在本文件追加 §3.x 子节收录。

---

## 1. 使用方式

1. IUT MUST 已通过 [NPS-Node-L1](./NPS-Node-L1.cn.md) —— L2 严格 additive。
2. 构建或安装 **被测实现**（**IUT**）。
3. 启动一个 **对端（peer）**—— 任意已通过 L2 的实现，或 .NET 参考 SDK
   （`impl/dotnet/src/NPS.NWP.Anchor/` 下的 `AnchorNodeClient`，CR-0002 §4）。
4. 将 IUT 与 peer 配对，逐条跑完 §3 所有测试用例。
5. 用例通过当且仅当 **全部** 验收条件成立。
6. §3 全部用例 MUST 通过，方可声明对应的 L2 要求（本文件即 L2-08）；不接受部分声明。
7. 将 [`NPS-NODE-L2-CERTIFIED.md`](./NPS-NODE-L2-CERTIFIED.md) 复制到 IUT 仓库根目录，
   填完每个字段，用 IUT 的 root 私钥对声明块签名。

paired-peer 方法论与 [NPS-Node-L1 §1](./NPS-Node-L1.cn.md) 一致。peer 要求升级：
凡 L1 要求"已通过 L1 的 peer"之处，L2 改为"已通过 L2 的 peer"。

本次发布接受自声明。第三方认证（NPS Cloud CA）规划于 2027 Q1+ 的 L3 阶段，不在本套件范围。

---

## 2. 测试环境

| 项 | 值 |
|----|----|
| 网络 | 仅 loopback；用例不要求外部 DNS 或跨机路由 |
| Peer | 任一 L2 合规实现（推荐：`NPS.NWP.Anchor` + `NPS.NDP` NPS v1.0.0-alpha.11 或更高） |
| 时钟 | 与 peer 墙钟偏差 ≤ ±5 秒（ISO 8601 UTC） |
| 文件系统 | IUT 密钥库与（如适用）拓扑持久化可写目录 |
| Wire 编码 | Tier-1 JSON MUST 覆盖；Tier-2 MsgPack SHOULD 覆盖 |
| 集群前置 | IUT MUST 充当单集群的 **Anchor Node**；成员节点由 peer 模拟 —— 通过 NDP `Announce` 帧携带 `cluster_anchor` = IUT 的 NID |
| 多 Anchor 前置 | 仅 §3.4（`TC-N2-HA-*`）需要：peer 额外模拟同一 `cluster_anchor` 集群下的一个或多个 **standby Anchor**；`TC-N2-HA-07` / `-08` 中 IUT 以 **NDP Registry** 角色而非 Anchor 角色受测 |

每个用例从 IUT 干净状态开始：用例间 MUST 清空 IUT 拓扑注册表，除非用例显式标注为 continuation。

---

## 3. 测试用例

每个用例列出 [NPS-AaaS-Profile §4.3](../NPS-AaaS-Profile.cn.md) 的 Req ID、前置条件、操作
步骤、验收条件。

### 3.1 Anchor 拓扑 —— `topology.snapshot` / `topology.stream`

本组用例验证 L2-08：实现 [NPS-2 §12](../../NPS-2-NWP.cn.md) 定义的保留查询类型。
TC-N2-AnchorTopo-01 至 TC-N2-AnchorStream-04 覆盖正向路径；TC-N2-AnchorTopo-04
至 TC-N2-AnchorTopo-08 覆盖必要的负路径（每个错误码对应一个 MUST 拒绝场景）。L2-08 需全部十二个用例通过方可声明合规。

#### TC-N2-AnchorTopo-01 —— 3 成员集群快照
**Req**：L2-08（`topology.snapshot`）
**前置**：IUT 充当 Anchor，无既有拓扑状态。
**操作**：
1. Peer 通过 NDP `Announce` 帧（`cluster_anchor` = IUT NID）声明三个成员节点（`M1` Memory、`M2` Action、`M3` Complex）。
2. Peer 等待 200 ms 让 IUT 完成摄入。
3. Peer 发送 `QueryFrame`，`type = "topology.snapshot"`、`topology.scope = "cluster"`、`topology.include` 取默认。
**通过条件**：
- IUT 用 `CapsFrame` 应答，`anchor_ref = "nps:system:topology:snapshot"`。
- 响应 payload 中 `cluster_size = 3`。
- `members` 数组恰好包含 `M1`、`M2`、`M3` 三个 NID（顺序不限）。
- 每个 member 对象至少含 `nid`、`node_roles`、`activation_mode`。
- `version` 是正整数。
- 响应携带的 `anchor_nid` 等于 IUT NID。

#### TC-N2-AnchorTopo-02 —— 入集群时 version 单调
**Req**：L2-08（version 语义，[NPS-2 §12.3](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，无既有拓扑状态。
**操作**：
1. Peer 声明成员 `M1`。
2. Peer 抓 `snapshot_a`（记下 `version = V1`）。
3. Peer 声明成员 `M2`。
4. Peer 抓 `snapshot_b`（记下 `version = V2`）。
**通过条件**：
- `V2 > V1` 严格成立。
- `snapshot_b.cluster_size = snapshot_a.cluster_size + 1`。
- IUT 的 `version` 计数器在本次运行内全程单调。

#### TC-N2-AnchorTopo-03 —— 子 Anchor 成员体现 `child_anchor` + `member_count`
**Req**：L2-08（子 Anchor 成员表示，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor；peer 模拟一个本身有 2 个成员的子 Anchor `CA`。
**操作**：
1. Peer 向 IUT 声明子 Anchor `CA`：`node_roles = ["anchor"]`，`cluster_anchor` = IUT NID，元数据指明 `CA` 自身有 2 个成员。
2. Peer 以默认 `topology.depth = 1` 抓 IUT 的快照。
**通过条件**：
- `CA` 对应的 member 对象 `child_anchor: true`。
- `CA` 对应的 member 对象 `member_count: 2`。
- 快照 `truncated` 字段缺省或为 `false`（深度 1 是默认值，未触及上限）。

#### TC-N2-AnchorStream-01 —— NDP Announce 触发 `member_joined`
**Req**：L2-08（`topology.stream` 加入事件，[NPS-2 §12.2](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，无既有拓扑状态。
**操作**：
1. Peer 通过 `SubscribeFrame` 订阅，`type = "topology.stream"`、`topology.scope = "cluster"`。
2. Peer 等待 `subscribed` 应答。
3. Peer 声明新成员 `M1`。
**通过条件**：
- announce 后 1 秒内，IUT 推送 `DiffFrame`，`event_type = "member_joined"`。
- 推送事件的 `payload` 是完整 member 对象，`nid` 与 `M1` 匹配。
- 事件的 `seq` 大于 subscribed 应答中返回的 `last_seq`。

#### TC-N2-AnchorStream-02 —— NDP TTL 到期触发 `member_left`
**Req**：L2-08（`topology.stream` 离开事件，[NPS-2 §12.2](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，已声明一个 TTL 较短（如 2 秒）的成员 `M1`。
**操作**：
1. Peer 订阅 `topology.stream`。
2. Peer 停止刷新 `M1` 的 announce。
3. Peer 等过 TTL。
**通过条件**：
- 最后一次刷新后 `TTL + 1 秒` 内，IUT 推送 `DiffFrame`，`event_type = "member_left"`。
- 推送事件的 `payload.nid` 与 `M1` 的 NID 匹配。
- 事件的 `seq` 严格大于上一次 `member_joined` 事件的 `seq`。

#### TC-N2-AnchorStream-03 —— `topology.since_version` 恢复重播
**Req**：L2-08（重播语义，[NPS-2 §12.2 / §12.3](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，无既有拓扑状态。
**操作**：
1. Peer 订阅 `topology.stream`，声明 `M1`。Peer 记录得到的 `seq = V1`。
2. Peer 声明 `M2`、`M3`（得到 `V2`、`V3`）。
3. Peer 断开订阅。
4. Peer 重新订阅，带 `topology.since_version = V1`。
**通过条件**：
- IUT 重播恰好两条事件：`M2` 与 `M3` 的 `member_joined`，顺序为 `V2`、`V3`。
- `M1`（即 `V1`）不被重播（`V1` 是边界，重播严格在其之后）。
- 不发出 `resync_required` 事件。

#### TC-N2-AnchorTopo-04 — 未鉴权拓扑访问（缺少 `topology:read`）→ `NWP-TOPOLOGY-UNAUTHORIZED`
**Req**: L2-08（鉴权门控，[NPS-2 §12.4](../../NPS-2-NWP.cn.md)；M6）
**Fixture**：IUT 作为 Anchor，已有一个已公告成员。
**Action**：
1. Peer 出示一个 IdentFrame，`capabilities` 中**不含** `topology:read`（其他必要能力均已包含）。
2. Peer 发送 `type = "topology.snapshot"` 的 `QueryFrame`。
**Pass**：
- IUT 响应一个携带错误码 `NWP-TOPOLOGY-UNAUTHORIZED` 的 `ErrorFrame`。
- IUT 不返回任何快照 payload 或部分成员数据。
- IUT 不得静默丢弃请求——MUST 发送错误响应。

#### TC-N2-AnchorTopo-05 — depth 超过上限 → `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`
**Req**: L2-08（depth 限制执行，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**Fixture**：IUT 作为 Anchor，最大 `topology.depth` 可通过配置指定；未另行指定时使用 `max_depth = 3`。
**Action**：
1. Peer（携带 `topology:read`）发送 `type = "topology.snapshot"` 且 `topology.depth = max_depth + 1`（例如 `max_depth = 3` 时请求 `4`）的 `QueryFrame`。
**Pass**：
- IUT 响应一个携带错误码 `NWP-TOPOLOGY-DEPTH-UNSUPPORTED` 的 `ErrorFrame`。
- IUT 不得在无错误的情况下静默截断或返回部分快照。

#### TC-N2-AnchorTopo-06 — 不识别的 `topology.scope` 值 → `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`
**Req**: L2-08（scope 校验，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**Fixture**：IUT 作为 Anchor，已有一个已公告成员。
**Action**：
1. Peer（携带 `topology:read`）发送 `type = "topology.snapshot"` 且 `topology.scope = "nonexistent_scope"` 的 `QueryFrame`。
**Pass**：
- IUT 响应一个携带错误码 `NWP-TOPOLOGY-UNSUPPORTED-SCOPE` 的 `ErrorFrame`。
- IUT 不得静默回退到默认 scope 并返回数据。

#### TC-N2-AnchorTopo-07 — 不识别的 `topology.filter` 键 → `NWP-TOPOLOGY-FILTER-UNSUPPORTED`
**Req**: L2-08（filter 键校验，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**Fixture**：IUT 作为 Anchor，已有一个已公告成员。
**Action**：
1. Peer（携带 `topology:read`）发送 `type = "topology.snapshot"` 且 `topology.filter = { "nonexistent_key": "value" }` 的 `QueryFrame`。
**Pass**：
- IUT 响应一个携带错误码 `NWP-TOPOLOGY-FILTER-UNSUPPORTED` 的 `ErrorFrame`。
- IUT 不得静默忽略未知键并返回未经过滤的数据。

#### TC-N2-AnchorTopo-08 — 不识别的保留 `type` 值 → `NWP-RESERVED-TYPE-UNSUPPORTED`
**Req**: L2-08（保留类型校验，[NPS-2 §12](../../NPS-2-NWP.cn.md)；M4）
**Fixture**：IUT 作为 Anchor。
**Action**：
1. Peer（携带 `topology:read`）发送 `type = "topology.nonexistent_operation"` 的 `QueryFrame`。
**Pass**：
- IUT 响应一个携带错误码 `NWP-RESERVED-TYPE-UNSUPPORTED` 的 `ErrorFrame`。
- IUT 不得响应 `NWP-ACTION-NOT-FOUND`（这两个错误码含义明确不同——见 NPS-2 §13）。
- IUT 不得静默忽略未知 type。

#### TC-N2-AnchorStream-04 —— version 过旧时 `resync_required`
**Req**：L2-08（`resync_required` 语义，[NPS-2 §12.2](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，保留环形缓冲配置为 5 个事件。
**操作**：
1. Peer 顺序声明 10 个成员（得到 `seq` 取值 `V1..V10`）。
2. Peer 订阅 `topology.stream`，带 `topology.since_version = 1`。
**通过条件**：
- IUT 推送的第一个事件 `event_type = "resync_required"`，`payload.reason = "version_too_old"`。
- 在 `resync_required` 事件之前不发送任何成员事件。
- Peer 可通过重发 `topology.snapshot` 后再无 `topology.since_version` 订阅来恢复。

### 3.2 NCP-over-TLS 入站终结（NPS-RFC-0006 §6）

以下用例适用于在 L2 入站网关（如 `nps-ingress`）终结原生模式 NCP-over-TLS 的 IUT，
验证 [NPS-RFC-0006 §6](../../rfcs/NPS-RFC-0006-ncp-native-transport.md) 与
NCP v0.8 §7.5 新增的传输安全准入门。

#### TC-N2-Tls-01 —— TLS 1.3 上协商 ALPN `nps/1.0`
**Req**：NPS-RFC-0006 §6.1–§6.2
**前置**：IUT 充当 L2 终结器，已配置服务器证书。
**操作**：
1. Peer 发起 TLS 1.3 连接，提供 ALPN `nps/1.0` 与有效客户端证书。
**通过条件**：
- 握手完成且协商出的 ALPN 为 `nps/1.0`。
- 未提供 / 提供未知 ALPN 的 Peer 以 TLS alert `no_application_protocol`（120）被拒。

#### TC-N2-Tls-02 —— 强制双向 TLS
**Req**：NPS-RFC-0006 §6.3（mTLS）
**前置**：IUT 配置 `RequireClientCert = true`。
**操作**：
1. Peer 不出示客户端证书尝试连接。
**通过条件**：
- IUT 拒绝该连接（任何 NCP 帧都不得转发给后端）。

#### TC-N2-Tls-03 —— 客户端证书验证到信任锚并绑定 NID
**Req**：NPS-RFC-0006 §6.3
**前置**：IUT 配置了信任锚；Peer 持有链到该信任锚的 NIP 叶证书。
**操作**：
1. Peer 携带有效客户端证书连接，并发送 preamble + HelloFrame + IdentFrame。
**通过条件**：
- 证书验证到信任锚；证书主体 NID 绑定到会话。
- 终结后的 NCP 字节流被原样代理到后端（preamble + 帧重放）。

#### TC-N2-Tls-04 —— IdentFrame 与证书 NID 不一致 → `NCP-NID-MISMATCH`
**Req**：NPS-RFC-0006 §6.3，错误码 `NCP-NID-MISMATCH`
**前置**：IUT 同上。
**操作**：
1. Peer 出示 NID 为 `urn:nps:agent:A` 的有效证书，但其 IdentFrame 声明 NID `urn:nps:agent:B`。
**通过条件**：
- IUT 以 `NCP-NID-MISMATCH` 关闭会话，且不把该 IdentFrame 转发给后端。

### 3.3 Bridge Node 入站（[NPS-2 §16.1.2](../../NPS-2-NWP.cn.md)、[NPS-CR-0010](../../cr/NPS-CR-0010-bridge-bidirectional.md)）

以下用例适用于声明了非空 `bridge_inbound_protocols`（NPS-4 §3.1）的 IUT ——
即主张入站 Bridge 合规 profile 的实现。既有的出站 `bridge_target` 往返向量
（[NPS-2 §16.4](../../NPS-2-NWP.cn.md)）仍是出站 profile 的套件，保持不变；
NPS-CR-0010 §4 把两个族分别称为 `TC-N2-BRIDGE-OUT-*` / `TC-N2-BRIDGE-IN-*`，
在本文档中分别落地为 §16.4 向量与 `TC-N2-BridgeIn-01..06`。

#### TC-N2-BridgeIn-01 —— MCP 入站提供完整必需方法集
**Req**：NPS-2 §16.1.2 MUST-3
**前置**：IUT 声明 `bridge_inbound_protocols: ["mcp"]`，前端一个 Memory Node 与一个 Action Node。
**操作**：
1. 一个纯 MCP 客户端（JSON-RPC 2.0，不带任何 NPS 库）依次调用：`initialize`、`ping`、
   `tools/list`、`tools/call`、`resources/list`、`resources/read`。
**通过条件**：
- 六个方法全部返回成功的 JSON-RPC 结果。
- `tools/list` 以限定名 `node__action` 暴露 Action Node 的 action（CR-0010 §5.1）。
- `resources/list` / `resources/read` 投影 Memory Node 的读表面；未前端 Memory Node 的 IUT
  也 MUST 服务这两个方法（返回空集），不得返回 "method not found"。

#### TC-N2-BridgeIn-02 —— gRPC 入站往返
**Req**：NPS-2 §16.1.2 MUST-2、MUST-3
**前置**：IUT 声明 `bridge_inbound_protocols: ["grpc"]`，前端一个 Action Node；客户端使用已发布的 `nwp_ingress.proto` 契约。
**操作**：
1. 一个纯 gRPC 客户端以有效 action id 和 JSON payload 调用一元 invoke RPC。
**通过条件**：
- 调用成功，响应 payload 等于该 Action Node 的 NWP 结果体。
- 客户端全程未提供任何 NID、帧或 NPS 寻址知识。

#### TC-N2-BridgeIn-03 —— A2A 入站往返
**Req**：NPS-2 §16.1.2 MUST-2、MUST-3
**前置**：IUT 声明 `bridge_inbound_protocols: ["a2a"]`，前端一个 Action Node。
**操作**：
1. 一个纯 A2A 客户端获取 AgentCard，然后对列出的 skill 提交 `tasks/send`。
**通过条件**：
- AgentCard 以限定名列出前端 action 作为 skill（CR-0010 §5.1）。
- `tasks/send` 完成并把 action 结果作为 task artifact 返回。

#### TC-N2-BridgeIn-04 —— 裸 action id 可解析、歧义必须拒绝
**Req**：NPS-CR-0010 §5.1（输入宽容）
**前置**：IUT 前端两个节点：恰好一个定义 action `orders_lookup`，两个都定义 action `status`。
**操作**：
1. 客户端以裸名 `orders_lookup` 调用 `tools/call`（MCP）。
2. 客户端以裸名 `status` 调用 `tools/call`。
**通过条件**：
- 调用 1 成功解析并执行（裸 id 无歧义）。
- 调用 2 以确定性的 JSON-RPC 错误被拒，错误信息列出两个限定名候选；MUST NOT 静默任选其一。

#### TC-N2-BridgeIn-05 —— 错误映射符合 §16.3 表
**Req**：NPS-2 §16.1.2 MUST-4、§16.3
**前置**：IUT 声明 `bridge_inbound_protocols: ["mcp"]`，前端一个可被制造失败的 Action Node。
**操作**：
1. 客户端分三次调用分别触发：鉴权失败、未知 action、上游超时。
**通过条件**：
- 每个失败都以 **JSON-RPC error** 呈现，错误码等于底层 NPS 状态在 §16.3 中的映射行 ——
  鉴权类错误绝不允许以 `isError: true` 的成功结果返回。
- 不同的 NPS 状态类映射到不同的外部错误码（§16.1.2 SHOULD-2 可观测性）。

#### TC-N2-BridgeIn-06 —— 未声明的协议/方向必须拒绝
**Req**：NPS-2 §16.1.2 MUST-5，错误码 `NWP-BRIDGE-DIRECTION-UNSUPPORTED`
**前置**：IUT 仅声明 `bridge_inbound_protocols: ["mcp"]`。
**操作**：
1. 客户端向 IUT 的入站表面发送一个格式良好的 A2A `tasks/send`。
**通过条件**：
- 请求以 `NWP-BRIDGE-DIRECTION-UNSUPPORTED` 被拒。
- 响应 `hint` SHOULD 同时携带两个已声明数组（`bridge_protocols`、`bridge_inbound_protocols`）。

### 3.4 多 Anchor 高可用（[NPS-2 §12.2](../../NPS-2-NWP.cn.md)、[NPS-4 §9](../../NPS-4-NDP.cn.md)、[NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md)）

本组用例落地 [NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md) §4 承诺的 `TC-N2-HA-*` 族，
验证该 CR 引入的集群所有权契约：`cluster_epoch` 围栏（[NPS-2 §12.2](../../NPS-2-NWP.cn.md)）、
两个已定稿的 `anchor_state` 子类型 `anchor_failover` / `anchor_quorum_lost`，以及最高 epoch
解析规则（[NPS-4 §9](../../NPS-4-NDP.cn.md)）。多 Anchor HA 的**运行**属于 AaaS Profile L3
要求；L1/L2 的单 Anchor 集群依然完全合规，由 TC-N2-HA-09 覆盖。

该契约横跨两种节点角色，因此 §4 把本族拆成三组认证：`TC-N2-HA-01..06` 以 **Anchor** 角色
考察 IUT；`TC-N2-HA-07..08` 以 **NDP Registry** 角色考察 IUT；`TC-N2-HA-09` 是第一组的
单 Anchor 镜像用例，与之互斥。

本小节的 fixture 约定：`C` 为集群稳定的 `cluster_anchor` NID；`A1` / `A2` 为该集群的 Anchor；
`E` 为当前活跃 owner 持有的 `cluster_epoch`。所有权如何取得、续租、交出属于实现自定义 ——
CR-0009 §1 刻意不规定共识传输 —— 因此每个用例只给出**可观测**的触发条件，IUT MUST 文档化
用于驱动该触发的控制面。

> **CR-0009 未定、在此为可测性钉住的两点**：(a) 入站帧携带**低于**接收方自身的
> `cluster_epoch` 不在 CR-0009 §3.1 覆盖范围内；本组用例对此不做任何断言，在后续 CR 定论前
> 实现 MAY 接受或拒绝。(b) CR-0009 §3.3 只说 quorum 恢复由"一个带全新 `cluster_epoch` 的
> 正常 `anchor_state` 事件"发出信号，未指明子类型，故 TC-N2-HA-04 只断言 epoch 刷新，
> 不断言具体的 `field` 取值。

#### TC-N2-HA-01 —— 两个拓扑读表面都携带 `cluster_epoch`
**Req**：NPS-CR-0009 §3.1（所有权围栏）、[NPS-2 §12.2](../../NPS-2-NWP.cn.md)
**前置**：IUT 充当集群 `C` 的**活跃** Anchor，持有 `cluster_epoch = E`；peer 模拟 standby `A2`；已公告一个成员。
**操作**：
1. Peer（携带 `topology:read`）发送 `type = "topology.snapshot"` 的 `QueryFrame`。
2. Peer 通过 `SubscribeFrame` 以 `type = "topology.stream"` 订阅，并读取 `subscribed` 应答。
**通过条件**：
- 快照响应携带 `cluster_epoch`，类型为 uint64 且 ≥ 1。
- `subscribed` 应答报告的 `cluster_epoch` 与快照一致。
- 两个值都等于 IUT 当前持有 `C` 所有权的 epoch；IUT 不得在两个表面报告不同的 epoch。

#### TC-N2-HA-02 —— 计划内交接时的 `anchor_failover` 线格式
**Req**：NPS-CR-0009 §3.2、[NPS-2 §12.2](../../NPS-2-NWP.cn.md) `anchor_state` 子类型 `anchor_failover`
**前置**：IUT 充当 `C` 的活跃 Anchor，`cluster_epoch = E`；peer 模拟的 standby `A2` 已就绪待接管。
**操作**：
1. Peer 订阅 `topology.stream` 并等待 `subscribed` 应答。
2. 操作方通过 IUT 已文档化的控制面触发从 IUT 到 `A2` 的**优雅**所有权交接。
**通过条件**：
- 1 秒内 IUT 推送 `DiffFrame`，`event_type = "anchor_state"`、`payload.field = "anchor_failover"`。
- `payload.details` 携带全部三个必填字段 `successor_nid`、`cluster_epoch`、`reason`。
- `successor_nid` 等于 `A2` 的 NID；`cluster_epoch` 为 uint64 且**严格大于** `E`；`reason = "planned"`。
- 该事件不得退化为裸的 `member_updated` 或 `resync_required` —— 必须使用带 `field` 判别符的 `anchor_state` 信封。

#### TC-N2-HA-03 —— 活跃方失联的 `anchor_failover` 是终结事件
**Req**：NPS-CR-0009 §3.2（被围栏的前任 leader MUST 发出一个终结性 `anchor_failover` 后关闭其流）
**前置**：IUT 充当 `C` 的活跃 Anchor，`cluster_epoch = E`，已有一个订阅者；peer 模拟的 `A2` 将在 IUT 租约失效后于 `E + 1` 接管所有权。
**操作**：
1. Peer 订阅 `topology.stream`。
2. 使 IUT 的所有权租约失效（例如将其租约对端网络隔离）；`A2` 于 `E + 1` 取得所有权，并让该事实对 IUT 可观测。
**通过条件**：
- IUT 推送 `anchor_state` 事件：`payload.field = "anchor_failover"`、`payload.details.reason = "active_lost"`、`successor_nid` 为 `A2` 的 NID、`cluster_epoch = E + 1`。
- 该事件是流上的**最后**一条事件：IUT 随后关闭订阅，不再推送任何拓扑事件。
- 交接后 IUT 不再表现为 owner —— 向其发送的拓扑写按 TC-N2-HA-05 被拒。
- IUT 不得不发终结事件就直接断开订阅。

#### TC-N2-HA-04 —— `anchor_quorum_lost` 线格式与降级只读运行
**Req**：NPS-CR-0009 §3.3、[NPS-2 §12.2](../../NPS-2-NWP.cn.md) `anchor_state` 子类型 `anchor_quorum_lost`
**前置**：IUT 充当三 Anchor 集群的活跃 Anchor，`quorum_size = 2`；已公告一个成员；已有一个订阅者。
**操作**：
1. Peer 订阅 `topology.stream`。
2. 使两个 peer 模拟的 Anchor 不可达，只剩 IUT（`available = 1`，低于 `quorum_size = 2`）。
3. Peer 发起一次 `topology.snapshot`（读），并另发起一次拓扑写。
4. 恢复两个 peer Anchor。
**通过条件**：
- IUT 推送 `DiffFrame`，`event_type = "anchor_state"`、`payload.field = "anchor_quorum_lost"`。
- `payload.details` 携带两个必填字段 `quorum_size` 与 `available`（均为 uint32），且 `quorum_size = 2`、`available = 1`，与 fixture 一致。
- 步骤 3 的读仍然成功（降级读被保留），且 IUT 自身的 NDP `AnnounceFrame` 报告 `health: "degraded"`（[NPS-4 §3.2](../../NPS-4-NDP.cn.md)）。
- 步骤 3 的写以 `NWP-ANCHOR-NOT-LEADER` 被拒。
- 恢复后 IUT 发出的 `anchor_state` 事件其 `cluster_epoch` 严格大于失去 quorum 前的 epoch；不得以旧 epoch 恢复接受写。

#### TC-N2-HA-05 —— standby Anchor 拒绝拓扑写 → `NWP-ANCHOR-NOT-LEADER`
**Req**：NPS-CR-0009 §3.1，错误码 `NWP-ANCHOR-NOT-LEADER`
**前置**：IUT 配置为 `C` 的 **standby** Anchor；peer 模拟的 Anchor `A1` 以 `cluster_epoch = E` 持有所有权。
**操作**：
1. Peer（持 `topology:read` 与写权限）向 IUT 推送一个会改变集群成员的拓扑写（`DiffFrame` / `AnnounceFrame`），携带 `cluster_epoch = E`。
2. Peer 随后向 IUT 发送 `topology.snapshot`。
**通过条件**：
- 步骤 1 被携带 `NWP-ANCHOR-NOT-LEADER` 的 `ErrorFrame` 拒绝，映射到 `NPS-CLIENT-CONFLICT`（HTTP 模式下为 409）。
- IUT 的拓扑不因该被拒的写而改变。
- IUT 不得静默把该写转发给 `A1` 并报告成功 —— standby MUST 拒绝，不得代理。
- 步骤 2 MAY 作为陈旧读成功；若成功，响应携带 IUT 自身最后已知的 `cluster_epoch`（≤ `E`）。standby MUST NOT 报告自己从未观测过的 epoch。

#### TC-N2-HA-06 —— 被取代的 leader 被围栏 → `NWP-ANCHOR-EPOCH-FENCED`
**Req**：NPS-CR-0009 §3.1，错误码 `NWP-ANCHOR-EPOCH-FENCED`
**前置**：IUT 充当 `C` 的活跃 Anchor，自认持有 `cluster_epoch = E`。
**操作**：
1. Peer 向 IUT 发送携带 `cluster_epoch = E + 1` 的入站拓扑帧 —— 即来自一个已在更高 epoch 下接管所有权的对端。
2. Peer 再发送一个除 `cluster_epoch = E` 外完全相同的帧。
**通过条件**：
- 步骤 1 被携带 `NWP-ANCHOR-EPOCH-FENCED` 的 `ErrorFrame` 拒绝（→ `NPS-CLIENT-CONFLICT`）。
- IUT 不得以 `NWP-ANCHOR-NOT-LEADER` 应答步骤 1 —— 两个错误码含义不同：`NOT-LEADER` 是"你写到了非 owner"，`EPOCH-FENCED` 是"接收方是被取代的 owner"。
- IUT 不得在拒绝前先施加步骤 1 帧的效果。
- 步骤 2（epoch 相等）不被围栏：只有**严格大于**的入站 `cluster_epoch` 才围栏接收方。

#### TC-N2-HA-07 —— Registry 解析到 `cluster_epoch` 最高的 Anchor
**Req**：NPS-CR-0009 §3.4、[NPS-4 §9](../../NPS-4-NDP.cn.md) 多 Anchor 集群解析
**前置**：IUT 充当 **NDP Registry**（参考实现：`nps-registry`），无既有条目。
**操作**：
1. Peer 公告 Anchor `A1`：`cluster_anchor = C`、`cluster_epoch = 4`。
2. Peer 公告 Anchor `A2`：`cluster_anchor = C`、`cluster_epoch = 7`；两条条目均存活且在 TTL 内。
3. Peer 对 `C` 发送 `ResolveFrame`。
4. Peer 以 `cluster_epoch = 4` 重新公告 `A1`（仍在心跳的陈旧 leader），再次解析 `C`。
5. Peer 为第二个集群 `C2` 公告一个**省略** `cluster_epoch` 的 Anchor `A3`，同时公告 `cluster_epoch = 2` 的 `A4`，然后解析 `C2`。
**通过条件**：
- 步骤 3 解析到 `A2` —— 存活条目中 `cluster_epoch` 最高者。
- 解析结果回显 `cluster_epoch = 7`。
- 步骤 4 仍解析到 `A2`：Registry MUST NOT 把集群 `C` 降级到 epoch 4（按集群单调）。
- 步骤 5 解析到 `A4`：省略 `cluster_epoch` 的公告按 epoch `1` 处理，因而输给任何大于 1 的显式 epoch。

#### TC-N2-HA-08 —— 同 epoch 脑裂 → `NDP-CLUSTER-SPLIT`
**Req**：NPS-CR-0009 §3.4，错误码 `NDP-CLUSTER-SPLIT`
**前置**：IUT 充当 NDP Registry，无既有条目。
**操作**：
1. Peer 公告 Anchor `A1`：`cluster_anchor = C`、`cluster_epoch = 5`。
2. Peer 公告 Anchor `A2`：`cluster_anchor = C`、`cluster_epoch = 5`；两条条目均存活且在 TTL 内。
3. Peer 对 `C` 发送 `ResolveFrame`。
4. Peer 让 `A1` 的条目超过 TTL 老化，再次解析 `C`。
**通过条件**：
- 步骤 3 以携带 `NDP-CLUSTER-SPLIT` 的 `ErrorFrame` 应答（→ `NPS-CLIENT-CONFLICT`）。
- IUT 不得任选其一，也不得返回部分或"尽力而为"的解析结果。
- 其他集群的解析不受影响 —— 脑裂范围仅限 `C`。
- 步骤 4 解析到 `A2` 并成功：只剩一条存活条目（或一侧被更高 epoch 取代）后，脑裂无需人工介入即自行消解。

#### TC-N2-HA-09 —— 单 Anchor 集群保持 `cluster_epoch = 1` 且不发任何 HA 事件
**Req**：NPS-CR-0009 §4 / §5（向后兼容）、[NPS-2 §12.2](../../NPS-2-NWP.cn.md)
**前置**：IUT 充当其集群**唯一**的 Anchor，未配置多 Anchor HA —— 即默认的 L1/L2 部署形态。
**操作**：
1. Peer 订阅 `topology.stream` 并抓一次 `topology.snapshot`。
2. Peer 按 TC-N2-AnchorTopo-01 与 TC-N2-AnchorStream-01/-02 的方式驱动一轮完整的成员变更（加入、更新、离开）。
3. Peer 发送一个完全**省略** `cluster_epoch` 的拓扑写 —— 模拟 alpha.17 之前的对端。
4. Peer 重启 IUT，重复步骤 1。
**通过条件**：
- 步骤 1 与步骤 4 的每个快照及 `subscribed` 应答，要么省略 `cluster_epoch`，要么报告 `cluster_epoch = 1`；该值全程不增长，跨重启亦然。
- 全程不推送 `payload.field` 为 `"anchor_failover"` 或 `"anchor_quorum_lost"` 的 `anchor_state` 事件。
- 步骤 3 的写被接受并按 epoch 1 处理 —— IUT MUST NOT 因缺少该可选字段而拒绝对端。
- 全程不发出 `NWP-ANCHOR-NOT-LEADER` 或 `NWP-ANCHOR-EPOCH-FENCED`。
- §3.1 的十二个用例对该 IUT 依然全部原样通过。

---

## 4. 结果清单

合规跑结产出一份 manifest（JSON）汇总每个用例的结果。manifest 嵌入到
[`NPS-NODE-L2-CERTIFIED.md`](./NPS-NODE-L2-CERTIFIED.md)：

```json
{
  "profile": "NPS-Node-L2",
  "profile_version": "0.5",
  "scope": ["L2-08"],
  "iut": {
    "name": "example-anchor",
    "version": "0.1.0",
    "nid": "urn:nps:node:example.com:anchor-01"
  },
  "peer": {
    "name": "nps-dotnet-reference",
    "version": "1.0.0-alpha.11"
  },
  "run": {
    "date": "2026-04-27T00:00:00Z",
    "environment": "linux-x64 / 1 vCPU / 1 GB"
  },
  "cases": [
    { "id": "TC-N2-AnchorTopo-01", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-02", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-03", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-04", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-05", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-06", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-07", "result": "pass" },
    { "id": "TC-N2-AnchorTopo-08", "result": "pass" },
    { "id": "TC-N2-AnchorStream-01", "result": "pass" },
    { "id": "TC-N2-AnchorStream-02", "result": "pass" },
    { "id": "TC-N2-AnchorStream-03", "result": "pass" },
    { "id": "TC-N2-AnchorStream-04", "result": "pass" },
    { "id": "TC-N2-Tls-01", "result": "pass" },
    { "id": "TC-N2-Tls-02", "result": "pass" },
    { "id": "TC-N2-Tls-03", "result": "pass" },
    { "id": "TC-N2-Tls-04", "result": "pass" },
    { "id": "TC-N2-BridgeIn-01", "result": "na" },
    { "id": "TC-N2-BridgeIn-02", "result": "na" },
    { "id": "TC-N2-BridgeIn-03", "result": "na" },
    { "id": "TC-N2-BridgeIn-04", "result": "na" },
    { "id": "TC-N2-BridgeIn-05", "result": "na" },
    { "id": "TC-N2-BridgeIn-06", "result": "na" },
    { "id": "TC-N2-HA-01", "result": "na" },
    { "id": "TC-N2-HA-02", "result": "na" },
    { "id": "TC-N2-HA-03", "result": "na" },
    { "id": "TC-N2-HA-04", "result": "na" },
    { "id": "TC-N2-HA-05", "result": "na" },
    { "id": "TC-N2-HA-06", "result": "na" },
    { "id": "TC-N2-HA-07", "result": "na" },
    { "id": "TC-N2-HA-08", "result": "na" },
    { "id": "TC-N2-HA-09", "result": "pass" }
  ],
  "summary": { "pass": 17, "fail": 0, "skip": 0, "na": 14 }
}
```

上面的示例是一个**单 Anchor** 且仅 Anchor 角色的 IUT：它通过 12 个拓扑用例与 4 个 TLS 用例，
不是 Bridge、不跑 Registry、也未声明多 Anchor HA —— 因此 `TC-N2-HA-01..08` 记 `na`，
只跑向后兼容用例 `TC-N2-HA-09`。

认证**按用例族授予，族内全有或全无**：

- **L2-08 拓扑**（`TC-N2-AnchorTopo-*`、`TC-N2-AnchorStream-*`）—— 任何 L2 主张都要求
  12 个用例全部 `pass`。本族没有可选用例：要么实现 [NPS-2 §12](../../NPS-2-NWP.cn.md)
  定义的 `topology.snapshot` / `topology.stream`，要么不实现。
- **NCP-over-TLS 入站**（`TC-N2-Tls-*`）—— 终结原生模式 NCP-over-TLS 的 IUT 要求 4 个
  全部 `pass`；否则记 `na`。
- **Bridge 入站**（`TC-N2-BridgeIn-*`）—— 声明了非空 `bridge_inbound_protocols` 的 IUT
  要求 6 个全部 `pass`；outbound-only 或非 Bridge IUT 记 `na`。
- **多 Anchor HA — Anchor 侧**（`TC-N2-HA-01`..`TC-N2-HA-06`）—— 声明多 Anchor HA 的 IUT
  （AaaS Profile L3 能力，[NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md) §4）要求
  6 个全部 `pass`；单 Anchor IUT 记 `na`。
- **多 Anchor HA — Registry 侧**（`TC-N2-HA-07`、`TC-N2-HA-08`）—— 实现 NDP Registry 解析面
  的 IUT 要求 2 个全部 `pass`；仅 Anchor 角色的 IUT 记 `na`。
- **单 Anchor 向后兼容**（`TC-N2-HA-09`）—— 单 Anchor IUT MUST `pass`；声明多 Anchor HA 的
  IUT 记 `na`。合法 manifest 中本用例与 Anchor 侧族恰有一方为 `na` —— 二者互斥，
  不得同时 `na`，也不得同时 `pass`。

某个族整体记 `na` 不阻塞其他族的认证；族内部分 `na` 属于 manifest 错误。

后续 CR 覆盖其余 L2 要求（L2-01..L2-07）时，上面的 `scope` 数组与
`summary` 总数会同步扩展。

---

## 5. 参考套件位置

| 语言 | 路径 | 状态 |
|------|------|------|
| .NET 10（xUnit）| `impl/dotnet/tests/NPS.Tests/Daemons/Npsd/AnchorTopologyConformanceTests.cs` | 与本 CR 同期落地 |
| Python | `impl/python/tests/conformance/node_l2/` | TODO（Phase 2）|
| TypeScript | `impl/typescript/tests/conformance/node-l2/` | TODO（Phase 2）|

参考套件的测试名 MUST 与上方 `TC-N2-*` ID 保持一致，确保跑结报告与 §4 manifest 1:1 对齐。

---

## 6. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.5 | 2026-08-01 | 新增 §3.4 **多 Anchor 高可用**（`TC-N2-HA-01..09`），落地 [NPS-CR-0009](../../cr/NPS-CR-0009-multi-anchor-ha.md) §4 承诺的 `TC-N2-HA-*` 族：两个拓扑读表面均携带 `cluster_epoch`、计划内交接与活跃方失联两种 `anchor_failover` 线格式（后者为终结事件并关闭流）、`anchor_quorum_lost` 线格式及降级只读运行与新 epoch 恢复、standby 写 → `NWP-ANCHOR-NOT-LEADER`、被取代 leader → `NWP-ANCHOR-EPOCH-FENCED`、NDP 最高 epoch 解析且不降级、同 epoch 脑裂 → `NDP-CLUSTER-SPLIT`、单 Anchor 在 `cluster_epoch = 1` 下的向后兼容。§2 增加"多 Anchor 前置"行；§4 manifest 现枚举四个用例族（12 拓扑 + 4 TLS + 6 bridge + 9 HA），并定义 HA Anchor 侧与单 Anchor 向后兼容用例的互斥关系。CR-0009 留白的两点（低 epoch 入站帧；quorum 恢复所用的 `anchor_state` 子类型）在 §3.4 中显式标注而非断言。 |
| 0.4 | 2026-07-23 | 新增 §3.3 **Bridge Node 入站**（`TC-N2-BridgeIn-01..06`），落地 [NPS-CR-0010](../../cr/NPS-CR-0010-bridge-bidirectional.md) §4 承诺的 `TC-N2-BRIDGE-IN-*` 族：MCP 完整方法集（含 `resources/*`）、gRPC + A2A 入站往返、裸名/限定名解析、§16.3 错误映射保真、`NWP-BRIDGE-DIRECTION-UNSUPPORTED` 拒绝。§4 manifest 枚举全部三个用例族（12 拓扑 + 4 TLS + 6 bridge），按族认证并定义 `na` 语义。同时补记缺失的 0.3 变更行。 |
| 0.3 | 2026-06-12 | （补记行 —— 该变更随 alpha.13 发布时未写变更历史。）新增 §3.2 **NCP-over-TLS 入站终结**（`TC-N2-Tls-01..04`），验证 NPS-RFC-0006 §6 准入门：ALPN `nps/1.0`、mTLS 强制、信任锚验证 + 会话 NID 绑定、IdentFrame/证书 NID 不一致时 `NCP-NID-MISMATCH`。 |
| 0.2 | 2026-05-01 | 新增 5 个负路径测试用例（TC-N2-AnchorTopo-04 至 -08），强制执行"每个 MUST 拒绝子句有一个失败路径 TC"标准：无 `topology:read` 的未鉴权访问（`NWP-TOPOLOGY-UNAUTHORIZED`）、depth 超过 Anchor 上限（`NWP-TOPOLOGY-DEPTH-UNSUPPORTED`）、不识别的 scope（`NWP-TOPOLOGY-UNSUPPORTED-SCOPE`）、不识别的 filter key（`NWP-TOPOLOGY-FILTER-UNSUPPORTED`）、不识别的 reserved type（`NWP-RESERVED-TYPE-UNSUPPORTED`）。TC 总数 7 → 12；认证通过阈值同步更新。修正 TC-N2-AnchorTopo-01 和 -03 中 `node_kind` → `node_roles`（M1 一致性）。 |
| 0.1 | 2026-04-27 | 初始草案：覆盖 L2-08（`topology.snapshot` / `topology.stream`，[NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md)）的 7 个用例。paired-peer 方法论沿用 L1。 |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
