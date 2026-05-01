[English Version](./NPS-Node-Profile.md) | 中文版

# NPS-Node Profile：节点侧合规性规范

**Status**: Draft
**Version**: 0.1
**Date**: 2026-04-24
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.5), NPS-3 (NIP v0.3), NPS-4 (NDP v0.4)

> 本规范是 [NPS-AaaS Profile](./NPS-AaaS-Profile.cn.md) 的姊妹篇。AaaS Profile 定义
> *服务* 该做什么来暴露 Agent 能力；本规范定义 *节点宿主*（daemon、内嵌 SDK、runtime）
> 该做什么来加入 NPS 网络。两份 Profile 正交：一个部署可以同时挂两个徽章，或只挂其一，
> 或都不挂。

---

## 1. 概述

### 1.1 范围与读者

两类读者：

- **实现者**：构建 NPS 节点的工程师 —— daemon、内嵌库、Agent runtime。
- **运营者**：把节点接到图谱的人 —— NeuronHub、MCP 宿主、headless Agent 舰队。

每一级回答一个问题：*我节点要做到 L{n}，需要满足什么？*

### 1.2 合规级别一览

三级对应激活谱系的三个位置：

- **L1 Basic（基础）** —— *被动*：节点接收并服务帧；Agent 随消息具现化。
- **L2 Interactive（交互）** —— *主动*：节点维护常驻 Agent 进程，订阅并响应。
- **L3 Autonomous（自主）** —— *自适应*：节点托管可 hibernate 的 Agent，按需唤醒，可具现化 ephemeral peer。

### 1.3 激活模型（关键承载）

L1+ 节点上的每个 Agent 都携带一个 `activation_mode`：

| 模式 | 含义 | Daemon 角色 |
|------|------|-----------|
| `ephemeral` | Agent 不常驻；消息到达时由 Agent SDK 或等价 runtime 动态具现化为新进程；处理完即销毁。 | 按需 spawn，完成后回收。 |
| `resident` | Agent 常驻长进程，订阅 inbox；daemon 通过长连接 push 入站帧。 | 仅转发入站帧，不 spawn。 |
| `hybrid` | Agent 常驻但空闲时 hibernate；入站帧触发唤醒。 | 唤醒进程后 push；跟踪空闲计时。 |

`activation_mode` MUST 出现在 L1+ 节点发布的每个 AnnounceFrame 中，是发送方选择 push
/ pull 方向的依据。wire 编码权威定义见 [NPS-4 §3.1](../NPS-4-NDP.cn.md#31-announceframe-0x30)。

---

## 2. 合规矩阵

| 维度 | L1 Basic | L2 Interactive | L3 Autonomous |
|------|----------|----------------|---------------|
| **定位** | 被动收发帧 | 订阅并响应 | 按需具现化 |
| **NCP 编码** | Tier-1 JSON MUST；Tier-2 OPTIONAL | + Tier-2 MsgPack MUST | + 长连接多路复用 MUST |
| **NIP** | Ed25519 签 / 验 MUST | + 信任链校验 MUST | + 动态 sub-NID 签发 MUST |
| **NDP** | Announce / Resolve（静态）MUST | + GraphFrame 订阅 MUST | + 按需注册 / 下线 MUST |
| **NWP** | ActionFrame pull MUST | + ActionFrame push + Subscribe MUST | + Query fan-out MUST |
| **NOP** | 不要求 | + TaskFrame / DelegateFrame MUST | + DAG 执行 + K-of-N 同步屏障 MUST |
| **激活模式** | 仅 `ephemeral`（pull 模式下的伪常驻） | `resident` MUST；其他 MAY | 三种模式全支持；默认 `ephemeral` |
| **可观测性** | 每帧本地日志 MUST | + Prometheus 风格 metrics MUST | + OpenTelemetry traces MUST |
| **认证** | `nps-node-l1` 自检通过 | + `nps-node-l2` 自检 | + `nps-node-l3` 自检 + 第三方审计（NPS Cloud CA, 2027 Q1+） |

"+" 前缀表示累进：L2 实现 MUST 满足 L1 所有行；L3 实现 MUST 满足 L2 所有行。

---

## 3. Level 1 — 基础合规

### 3.1 NCP — Wire 格式

| Req ID | 描述 |
|--------|------|
| N1-NCP-01 | MUST 对 L1 涉及的每个 Tier-1 JSON 帧（HelloFrame、AnchorFrame、IdentFrame、AnnounceFrame、ResolveFrame、ActionFrame、CapsFrame、ErrorFrame）正确编解码。 |
| N1-NCP-02 | MUST 能与任一合规对端完成 Hello + Anchor 握手。 |
| N1-NCP-03 | MUST 在可配置地址上监听；默认 `127.0.0.1:17433`（loopback）。跨机监听在 L1 为 OPTIONAL。 |
| N1-NCP-04 | MAY 实现 Tier-2 MsgPack；如实现，MUST 经由 HelloFrame 协商。 |

### 3.2 NIP — 身份

| Req ID | 描述 |
|--------|------|
| N1-NIP-01 | MUST 首次运行时生成并持久化一把 Ed25519 root keypair，文件权限 `0600` 或平台等价。 |
| N1-NIP-02 | MUST 用 root 私钥对 IdentFrame 签名；每个入站连接 MUST 验 IdentFrame。 |
| N1-NIP-03 | MUST 暴露形如 `urn:nps:node:<authority>:<id>` 的稳定 NID，`<authority>` 为运营方域名或组织标识符，`<id>` 为每宿主稳定标识符。 |
| N1-NIP-04 | MAY 向托管 Agent 签发 session sub-NID（L3 MUST）。 |

### 3.3 NDP — 发现

| Req ID | 描述 |
|--------|------|
| N1-NDP-01 | MUST 在 AnnounceFrame 中携带 `activation_mode = "ephemeral"`（L1 仅支持 `ephemeral`）。 |
| N1-NDP-02 | MUST 用发布者 IdentFrame 私钥对 AnnounceFrame 签名。 |
| N1-NDP-03 | MUST 对本地注册表内的任一 Agent 应答 ResolveFrame。 |
| N1-NDP-04 | MAY 订阅 GraphFrame 更新（L2 MUST）。 |

### 3.4 NWP — Inbox 与交付

| Req ID | 描述 |
|--------|------|
| N1-NWP-01 | MUST 为每个 NID 维护一个 inbox，接收 ActionFrame。 |
| N1-NWP-02 | MUST 在 daemon 重启后保留未投递的 inbox 内容。 |
| N1-NWP-03 | MUST 通过 NWP pull 向 Agent 提供 inbox 内容（Agent 按需拉取）。 |
| N1-NWP-04 | MUST 在常规硬件上处理至少 100 QPS 的 pull 请求（baseline，不等于压测）。 |
| N1-NWP-05 | MAY 通过长连接 push 帧（L2 MUST）。 |

### 3.5 NOP — 编排

L1 不要求。L1 节点收到 TaskFrame MAY 返回 `NPS-CLIENT-NOT-IMPLEMENTED`。

### 3.6 可观测性

| Req ID | 描述 |
|--------|------|
| N1-OBS-01 | MUST 对收发的每个帧写一条结构化日志。 |
| N1-OBS-02 | 每条日志 MUST 包含：ISO 8601 UTC 时间戳、方向（`in`/`out`）、帧类型、源 NID、目的 NID、帧字节数。 |
| N1-OBS-03 | 日志目标由实现决定（文件 / stdout / journal）；L1 允许仅本地存储。 |

### 3.7 合规认证

通过 [`nps-node-l1`](./conformance/NPS-Node-L1.cn.md) 测试套件即获 L1 认证。通过者 MAY
将 `NPS-NODE-L1-CERTIFIED.md` 模板（见 conformance 目录）放到自己 repo 根作为公开声明。

---

## 4. Level 2 — 交互合规

L2 = L1 + §2 L2 列每个 "+" 行。要点：

- **NCP**：Tier-2 MsgPack MUST。
- **NIP**：对配置的信任锚做信任链校验 MUST。
- **NDP**：GraphFrame 订阅 MUST；节点 SHOULD 在 GraphFrame `seq` 窗口内响应增量变更。
- **NWP**：ActionFrame push 和 Subscribe MUST；pull 仍然可用。
- **NOP**：MUST 接受 TaskFrame、DelegateFrame；完整 DAG 执行在 L3。
- **激活**：MUST 支持 `resident` 模式；`hybrid` MAY 支持。
- **可观测性**：MUST 暴露 Prometheus 风格 metrics 端点（帧计数、inbox 深度、连接数、握手延迟直方图）。

详细 req ID（`N2-NCP-*` 等）TODO，追踪于[路线图 Phase 2](../../docs/roadmap.cn.md)。

---

## 5. Level 3 — 自主合规

L3 = L2 + §2 L3 列每个 "+" 行。要点：

- **NCP**：单条 TCP/TLS 连接 MUST 支持多路复用（单 socket 上并发多 Agent）。
- **NIP**：节点 MUST 能按需向托管 Agent 签发 session sub-NID，用 root 私钥签名，并能吊销。
- **NDP**：MUST 支持由托管 Agent 的 spawn / shutdown 触发的按需 register / deregister。
- **NWP**：MUST 支持 Query fan-out（一次入站查询 → 多个 Memory/Action 目标）。
- **NOP**：MUST 支持完整 DAG 执行与 K-of-N 同步屏障。
- **激活**：MUST 支持三种模式；新 Agent 默认 `ephemeral`。
- **可观测性**：MUST 对每个跨节点跳数发出 OpenTelemetry trace（W3C `traceparent` 传播）。

L3 引入 **spawn-on-demand** 要求。节点 MUST 能从 `spawn_spec_ref`（见 NPS-4 §3.1）在没有
已存在进程的情况下具现化一个 `ephemeral` Agent 进程；被具现化的进程 MUST 在实现定义的
启动预算内收到目标帧（建议：headless Agent ≤ 2 s）。`spawn_spec_ref` 内容 schema 在 L3 由
未来的姊妹文档（NPS-Daemon-Spec）标准化。

详细 req ID（`N3-*`）TODO，追踪于[路线图 Phase 3](../../docs/roadmap.cn.md)。

---

## 6. NDP Wire Δ（informative）

本规范依赖 NPS-4 NDP v0.4 AnnounceFrame (0x30) 的三个新增字段：

| 字段 | 必填 | 用于 |
|------|------|------|
| `activation_mode` | L1+ 发布方 REQUIRED；接收方 OPTIONAL（缺失时按 `ephemeral` 处理，与 NPS v1.0-alpha.2 节点向后兼容） | 所有级别 |
| `activation_endpoint` | `resident` / `hybrid` 发布方 REQUIRED | L2、L3 |
| `spawn_spec_ref` | OPTIONAL；对 `ephemeral` 和 `hybrid` 冷启动有意义 | L3 |

字段权威定义、JSON 示例、向后兼容规则在 [NPS-4 §3.1](../NPS-4-NDP.cn.md#31-announceframe-0x30)
和 [`spec/frame-registry.yaml`](../frame-registry.yaml) 中。

---

## 7. 合规认证

### 7.1 测试套件

| 级别 | 文档 | 参考套件 |
|------|------|---------|
| L1 | [`conformance/NPS-Node-L1.cn.md`](./conformance/NPS-Node-L1.cn.md) | .NET 10 / xUnit（本次发布） |
| L2 | `conformance/NPS-Node-L2.cn.md` | TODO（路线图 Phase 2） |
| L3 | `conformance/NPS-Node-L3.cn.md` | TODO（路线图 Phase 3） |

### 7.2 自声明

通过者 MAY 将 `NPS-NODE-L{n}-CERTIFIED.md` 模板（见 conformance 目录）复制到 repo 根作为
公开合规声明。

### 7.3 第三方认证

第三方认证（NPS Cloud CA）规划于 2027 Q1+ 的 L3 阶段，本次发布不提供。

---

## 8. 与 NPS-AaaS Profile 的关系

| Profile | 回答的问题 |
|---------|----------|
| NPS-AaaS | "我的服务是合规的 Agent-as-a-Service 提供方吗？" |
| NPS-Node | "我的宿主是 NPS 网络的合规参与方吗？" |

一个部署可同时挂两个徽章，或只挂其一，或都不挂。AaaS L1 不要求底层宿主 Node L1，但实操上，
AaaS Anchor Node 运营方 SHOULD 也为承载 Anchor 的 daemon 做 Node L1（这是 NPS-CR-0001 之前称为 "Gateway Node" 的角色）。

---

## 9. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1 | 2026-04-24 | 初稿：三级节点合规（L1/L2/L3）、`activation_mode` 模型、L1 详细 req ID、L1 conformance 套件引用、与 AaaS Profile 的关系 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
