[English Version](./NPS-Node-L2.md) | 中文版

# NPS-Node-L2 合规测试套件

**Status**: Draft
**Version**: 0.2
**Date**: 2026-05-01
**Applies-To**: [NPS-AaaS-Profile §4.3](../NPS-AaaS-Profile.cn.md) — Level 2 Standard
**Authors**: Ori Lynn / INNO LOTUS PTY LTD

> 本文档定义 Anchor Node 实现声明 NPS-AaaS-Profile L2 合规所 MUST 通过的测试用例，
> 当前版本仅覆盖 [NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md) 引入的
> **L2-08 拓扑读取** 要求。
>
> 其余 L2 要求（L2-01 至 L2-07 —— NOP 编排、OpenTelemetry 追踪、NPT Token Budget、
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
| Peer | 任一 L2 合规实现（推荐：`NPS.NWP.Anchor` + `NPS.NDP` NPS v1.0.0-alpha.4 或更高） |
| 时钟 | 与 peer 墙钟偏差 ≤ ±5 秒（ISO 8601 UTC） |
| 文件系统 | IUT 密钥库与（如适用）拓扑持久化可写目录 |
| Wire 编码 | Tier-1 JSON MUST 覆盖；Tier-2 MsgPack SHOULD 覆盖 |
| 集群前置 | IUT MUST 充当单集群的 **Anchor Node**；成员节点由 peer 模拟 —— 通过 NDP `Announce` 帧携带 `cluster_anchor` = IUT 的 NID |

每个用例从 IUT 干净状态开始：用例间 MUST 清空 IUT 拓扑注册表，除非用例显式标注为 continuation。

---

## 3. 测试用例

每个用例列出 [NPS-AaaS-Profile §4.3](../NPS-AaaS-Profile.cn.md) 的 Req ID、前置条件、操作
步骤、验收条件。

### 3.1 Anchor 拓扑 —— `topology.snapshot` / `topology.stream`

本组用例验证 L2-08：实现 [NPS-2 §12](../../NPS-2-NWP.cn.md) 定义的保留查询类型。
全部 12 个用例 MUST 通过方可声明 L2-08：
TC-N2-AnchorTopo-01 至 TC-N2-AnchorStream-04 覆盖 happy path；TC-N2-AnchorTopo-04
至 TC-N2-AnchorTopo-08 覆盖必填负路径（每个错误码一条 MUST-reject 用例）。

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

#### TC-N2-AnchorTopo-04 —— 无 `topology:read` 未鉴权访问 → `NWP-TOPOLOGY-UNAUTHORIZED`
**Req**：L2-08（鉴权门控，[NPS-2 §12.4](../../NPS-2-NWP.cn.md)；M6）
**前置**：IUT 充当 Anchor，已声明一个成员。
**操作**：
1. Peer 出示 `IdentFrame`，`capabilities` 中**不含** `topology:read`（其余所需 capability 均存在）。
2. Peer 发送 `QueryFrame`，`type = "topology.snapshot"`。
**通过条件**：
- IUT 用 `ErrorFrame` 应答，错误码为 `NWP-TOPOLOGY-UNAUTHORIZED`。
- IUT 不返回任何快照 payload 或部分成员数据。
- IUT 不静默丢弃请求——MUST 发出错误响应。

#### TC-N2-AnchorTopo-05 —— 超过深度上限 → `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`
**Req**：L2-08（深度约束，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，其最大 `topology.depth` 已记录或可配置；未指定时使用 `max_depth = 3`。
**操作**：
1. Peer（持有 `topology:read`）发送 `QueryFrame`，`type = "topology.snapshot"`，`topology.depth = max_depth + 1`（如 `max_depth = 3` 则取 `4`）。
**通过条件**：
- IUT 用 `ErrorFrame` 应答，错误码为 `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`。
- IUT 不静默截断或在不报错情况下返回部分快照。

#### TC-N2-AnchorTopo-06 —— 不识别的 `topology.scope` 值 → `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`
**Req**：L2-08（scope 校验，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，已声明一个成员。
**操作**：
1. Peer（持有 `topology:read`）发送 `QueryFrame`，`type = "topology.snapshot"`，`topology.scope = "nonexistent_scope"`。
**通过条件**：
- IUT 用 `ErrorFrame` 应答，错误码为 `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`。
- IUT 不静默回退到默认 scope 并返回数据。

#### TC-N2-AnchorTopo-07 —— 不识别的 `topology.filter` 键 → `NWP-TOPOLOGY-FILTER-UNSUPPORTED`
**Req**：L2-08（filter key 校验，[NPS-2 §12.1](../../NPS-2-NWP.cn.md)）
**前置**：IUT 充当 Anchor，已声明一个成员。
**操作**：
1. Peer（持有 `topology:read`）发送 `QueryFrame`，`type = "topology.snapshot"`，`topology.filter = { "nonexistent_key": "value" }`。
**通过条件**：
- IUT 用 `ErrorFrame` 应答，错误码为 `NWP-TOPOLOGY-FILTER-UNSUPPORTED`。
- IUT 不静默忽略未知 key 并返回未过滤数据。

#### TC-N2-AnchorTopo-08 —— 不识别的保留 `type` 值 → `NWP-RESERVED-TYPE-UNSUPPORTED`
**Req**：L2-08（保留类型校验，[NPS-2 §12](../../NPS-2-NWP.cn.md)；M4）
**前置**：IUT 充当 Anchor。
**操作**：
1. Peer（持有 `topology:read`）发送 `QueryFrame`，`type = "topology.nonexistent_operation"`。
**通过条件**：
- IUT 用 `ErrorFrame` 应答，错误码为 `NWP-RESERVED-TYPE-UNSUPPORTED`。
- IUT 不以 `NWP-ACTION-NOT-FOUND` 应答（两码明确区分——见 NPS-2 §13）。
- IUT 不静默忽略未知 type。

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

---

## 4. 结果清单

合规跑结产出一份 manifest（JSON）汇总每个用例的结果。manifest 嵌入到
[`NPS-NODE-L2-CERTIFIED.md`](./NPS-NODE-L2-CERTIFIED.md)：

```json
{
  "profile": "NPS-Node-L2",
  "profile_version": "0.1",
  "scope": ["L2-08"],
  "iut": {
    "name": "example-anchor",
    "version": "0.1.0",
    "nid": "urn:nps:node:example.com:anchor-01"
  },
  "peer": {
    "name": "nps-dotnet-reference",
    "version": "1.0.0-alpha.4"
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
    { "id": "TC-N2-AnchorStream-04", "result": "pass" }
  ],
  "summary": { "pass": 12, "fail": 0, "skip": 0, "na": 0 }
}
```

L2-08 的认证在 **全部 12 个用例 `pass`** 时授予。本范围下没有可选用例 ——
要么实现 [NPS-2 §12](../../NPS-2-NWP.cn.md) 定义的 `topology.snapshot` / `topology.stream`，
要么不实现。

后续 CR 在本文档追加 §3.2 起的子节覆盖 L2-01..L2-07 时，上面的 `scope` 数组与
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
| 0.2 | 2026-05-01 | 新增 5 个负路径测试用例（TC-N2-AnchorTopo-04 至 -08）以落实"每个 MUST-reject 子句须有失败路径 TC"标准：无 `topology:read` 的未鉴权访问（M6 capability gate，`NWP-TOPOLOGY-UNAUTHORIZED`）、depth 超过 Anchor 上限（`NWP-TOPOLOGY-DEPTH-UNSUPPORTED`）、不识别的 scope（`NWP-TOPOLOGY-UNSUPPORTED-SCOPE`）、不识别的 filter key（`NWP-TOPOLOGY-FILTER-UNSUPPORTED`）、不识别的 reserved type（`NWP-RESERVED-TYPE-UNSUPPORTED`）。TC 总数 7 → 12。修正 TC-N2-AnchorTopo-01 和 -03 中 `node_kind` → `node_roles`（M1 一致性）。 |
| 0.1 | 2026-04-27 | 初始草案：覆盖 L2-08（`topology.snapshot` / `topology.stream`，[NPS-CR-0002](../../cr/NPS-CR-0002-anchor-topology-queries.md)）的 7 个用例。paired-peer 方法论沿用 L1。 |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
