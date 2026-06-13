[English Version](./NPS-Node-L3.md) | 中文版

# NPS-Node-L3 合规测试套件

**状态**: 草案 (Draft)
**版本**: 0.1
**日期**: 2026-06-12
**适用于**: [NPS-Node-Profile](../NPS-Node-Profile.md) —— Level 3（按需 / FaaS）、[NPS-CR-0007](../../cr/NPS-CR-0007-nop-l3-runtime-integration.md)
**作者**: Ori Lynn / INNO LOTUS PTY LTD

> 本文件定义一个 **L3 运行时**（`nps-runner`）实现为声明 NPS-Node-Profile L3 合规所必须通过的测试用例，
> 覆盖 [NPS-CR-0007](../../cr/NPS-CR-0007-nop-l3-runtime-integration.md) 引入、并在
> [NPS-5-NOP §8](../../NPS-5-NOP.md) 中归纳的 runner ↔ NOP 编排接口。
>
> L3 在 NOP（NPS-5）及 L1/L2 节点套件已覆盖的编排语义之上严格增量，新增**任务认领协议**、
> **`spawn_spec_ref` 解析**与 **idle/max-runtime 生命周期强制**。

---

## 1. 如何使用本文件

1. 被测实现（IUT）是一个从每-NID 收件箱认领 `TaskFrame` 并实例化临时 worker 的 L3 运行时。
2. 每个 `TC-N3-*` 用例列出前置条件、动作与要求的可观测结果。
3. L3 自声明需全部用例通过（参见未来的 `NPS-NODE-L3-CERTIFIED.md`）。

---

## 2. 测试用例

### 2.1 任务认领协议（NPS-CR-0007 §4）

| 用例 | 前置条件 | 动作 | 要求结果 |
|------|----------|------|----------|
| **TC-N3-Claim-01** | 收件箱中有一个 `TaskFrame` | 两个 runner 并发认领 | 恰好一个**获授**；另一个返回 `NOP-CLAIM-CONFLICT`（HTTP 409）。无重复执行。 |
| **TC-N3-Claim-02** | 某任务处于 `LEASED` 且租约已过期 | 第二个 runner 重新认领；该节点已上报终态 | 认领**获授**（reclaim）；已处终态的节点**不**重复执行（`dedup_key` 命中）；从首个非终态节点恢复。 |
| **TC-N3-Claim-03** | 一个持有活跃租约的运行中任务 | 持有方 runner 在过期前续约 | 租约过期被延长；不对持有方触发 `NOP-CLAIM-CONFLICT`。 |

### 2.2 `spawn_spec_ref` 解析（NPS-CR-0007 §5）

| 用例 | 前置条件 | 动作 | 要求结果 |
|------|----------|------|----------|
| **TC-N3-Spawn-01** | AnnounceFrame 携带内联 `spawnspec:` base64url-JSON 的 `spawn_spec_ref` | runner 解析并 spawn | 由 SpawnSpec 的 `image`/`command`/`env`/`resource_limits` 构造 worker。 |
| **TC-N3-Spawn-02** | `spawn_spec_ref` 是返回合法 SpawnSpec JSON 的 `https://`/`nwp://` URL | runner 拉取并 spawn | 构造 worker；拉取的报文体经 schema 校验。 |
| **TC-N3-Spawn-03** | `spawn_spec_ref` 无法解析或 schema 校验失败 | runner 尝试解析 | `NOP-SPAWN-SPEC-INVALID`（HTTP 400）；节点 `FAILED`。 |

### 2.3 生命周期强制（NPS-CR-0007 §6）

| 用例 | 前置条件 | 动作 | 要求结果 |
|------|----------|------|----------|
| **TC-N3-Life-01** | worker 设 `idle_timeout_seconds = T` | worker 空闲超过 `T` 仍未完成节点 | `NOP-RUNTIME-IDLE-TIMEOUT`（HTTP 504）；节点 `FAILED`；worker 被回收。 |
| **TC-N3-Life-02** | worker 设 `max_runtime_seconds = T` | worker 运行超过 `T` 仍未完成 | `NOP-RUNTIME-MAX-RUNTIME`（HTTP 504）；节点 `FAILED`；worker 被回收。 |

### 2.4 端到端与 Saga

| 用例 | 前置条件 | 动作 | 要求结果 |
|------|----------|------|----------|
| **TC-N3-DAG-01** | 收件箱中有一个三节点线性 DAG TaskFrame | runner 认领、经 NDP `ResolveFrame` 解析 worker NID、spawn 并派发 `DelegateFrame` | 三节点端到端执行；终态 PATCH 到编排器任务存储；TaskFrame `COMPLETED`。 |
| **TC-N3-Saga-01** | DAG 中某 L3 节点带 `compensate_action`，且下游节点 FAILED | runner 运行 DAG；下游节点失败 | 逆拓扑 saga 回滚（NPS-5 §3.1.6）补偿已完成的 L3 前驱；状态 `COMPENSATING → COMPENSATED`。 |

---

## 3. 超出范围

分布式租约存储后端（多副本协调）属实现关切；本套件仅校验认领/spawn/生命周期契约的**可观测线上行为**，该行为与存储无关（NPS-CR-0007 §10 OQ-1）。

---

*版权所有：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
