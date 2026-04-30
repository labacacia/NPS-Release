[English Version](./NPS-AaaS-Profile.md) | 中文版

# NPS-AaaS Profile: Agent-as-a-Service 合规性规范

**Status**: Proposed
**Version**: 0.4
**Date**: 2026-04-27
**Authors**: Ori Lynn / INNO LOTUS PTY LTD
**Depends-On**: NPS-1 (NCP v0.6)、NPS-2 (NWP v0.8)、NPS-3 (NIP v0.4)、NPS-4 (NDP v0.5)、NPS-5 (NOP v0.4)

> 本规范定义基于 NPS 协议族构建 Agent-as-a-Service（AaaS）平台的合规性要求，
> 涵盖服务入口、内部编排、数据访问三层架构。

---

## 1. 概述

### 1.1 什么是 AaaS

Agent-as-a-Service 是一种云服务模型：服务提供商将 AI Agent 能力以标准化 API
暴露给消费方（其他 Agent 或人类应用），消费方无需了解内部实现细节。

### 1.2 NPS-AaaS 解决什么问题

| 现状痛点 | NPS-AaaS 方案 |
|----------|--------------|
| 各 Agent 平台 API 互不兼容 | 统一 Anchor Node 入口（集群前门）+ Bridge Node（NPS↔外部）+ NWP 标准帧协议 |
| 内部编排无标准、不可观测 | NOP DAG 编排 + OpenTelemetry 链路追踪 |
| AI 访问传统数据库 token 开销大 | Vector Proxy Layer 向量化中间层 |
| 无 Agent 身份/权限标准 | NIP NID 身份 + scope 委托链 |
| 服务质量无法保证 | NPT Token Budget + 背压控制 |

### 1.3 架构总览

```
Consumer Agent
      │
      ▼
┌─────────────────────────────────┐
│  Anchor Node（NWP 节点类型）     │  ← 服务入口，对外 API
│  • 认证 (NIP)                   │
│  • 路由 / 限流 / Token Budget   │
│  • 服务目录 (NWM)               │
└──────────┬──────────────────────┘
           │ DelegateFrame (NOP)
           ▼
┌─────────────────────────────────┐
│  Orchestration Layer (NOP)      │  ← 业务协调
│  • DAG 任务分解                  │
│  • K-of-N 同步 / 预检            │
│  • 重试 / 超时 / 取消            │
└──────┬────────────┬─────────────┘
       │            │
       ▼            ▼
┌────────────┐ ┌────────────────────┐
│ Action Node│ │ Memory Node        │
│ (Worker)   │ │ + Vector Proxy     │  ← 传统 DB 向量化
└────────────┘ │   Layer            │
               └────────────────────┘
```

---

## 2. Anchor Node（NWP 扩展）

> **由 [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md) 在 v1.0-alpha.3 从 "Gateway Node" 重命名而来。** 本节描述的角色 —— 把入站 NWP `Action` 帧无状态地路由为 NOP `TaskFrame` 的集群入口 —— 不变。Wire 值 `node_type: "gateway"` 已移除，解析器 MUST 拒绝。实现 MAY 在按请求无状态分派之上叠加成员节点的长期注册表（NPS-2 §2.1 *Anchor Node — 详细语义*）；拓扑读取接口（`topology.snapshot` / `topology.stream`）由 [NPS-2 §12](../NPS-2-NWP.cn.md) 定义，并在 L2 强制要求（见下文 §4.3），详见 [NPS-CR-0002](../cr/NPS-CR-0002-anchor-topology-queries.md)。

### 2.1 定义

**Anchor Node** 是 NWP 节点类型，作为 AaaS 服务的统一入口。
它不直接处理业务逻辑，而是将请求路由到内部 NOP 编排层。同一个 Anchor Node MAY 通过 NDP `Announce.node_kind` 数组形式同时声明其他角色（如 `Memory Node`），见 NPS-4 §3.1。

**与 Complex Node 的区别**：

| 属性 | Complex Node | Anchor Node |
|------|-------------|-------------|
| 业务逻辑 | 自身处理，可含计算 | **无**，仅路由到 NOP |
| 内部状态 | 有（数据+操作混合） | **无状态**（身份验证+路由），可选附加成员注册表 |
| Action 处理 | 直接执行 | 全部转换为 NOP TaskFrame |
| 子节点引用 | 可选 | 必须（NOP Worker 节点） |
| 适用场景 | 混合型业务节点 | 服务入口、集群路由 |

Anchor Node 可以视为一个「纯路由 Complex Node」——它的所有 Action 映射到
NOP DAG，自身不持有业务状态，因此可以水平扩展且无需关心副作用幂等。

| 属性 | 值 |
|------|------|
| 节点类型 | `anchor` |
| NWM node_type | `"anchor"` |
| NDP node_kind | `"anchor"`（或包含 `"anchor"` 的数组形式）|
| 帧入口 | ActionFrame (0x11) |
| 内部转换 | ActionFrame → NOP TaskFrame (0x40) |

### 2.2 Anchor Node 职责

| 职责 | 对应协议 | 描述 |
|------|---------|------|
| **身份验证** | NIP | 验证消费方 NID，校验 scope |
| **服务目录** | NWP NWM | 通过 NWM manifest 暴露可用 Action 列表 |
| **请求路由** | NOP | 将 ActionFrame 转换为 TaskFrame，分解 DAG |
| **Token 计量** | NPT | 请求级 Token Budget 管控 |
| **限流** | NWP | 基于 NID 的速率限制 |
| **可观测性** | NOP Context | 注入 trace_id/span_id，全链路追踪 |
| **集群注册表** *(L1 可选，L2 强制)* | NDP + NWP | 跟踪通过 `Announce.cluster_anchor` 注册的成员节点；通过 `topology.snapshot` / `topology.stream`（NPS-2 §12）暴露 |

### 2.3 NWM Anchor 清单示例

```json
{
  "nwm_version": "0.7",
  "node_type": "anchor",
  "node_id": "nwp://api.example.com/agent-service",
  "display_name": "Example AaaS Anchor",
  "capabilities": ["nop:orchestrate", "nwp:invoke", "nip:delegate"],
  "actions": [
    {
      "action_id": "analysis.run",
      "description": "Run a multi-step data analysis pipeline",
      "params_schema": { "$ref": "#/schemas/analysis_input" },
      "result_schema": { "$ref": "#/schemas/analysis_output" },
      "estimated_npt": 2000,
      "timeout_ms": 120000,
      "async": true
    }
  ],
  "rate_limits": {
    "requests_per_minute": 60,
    "max_concurrent": 10,
    "npt_per_hour": 100000
  },
  "auth": {
    "required": true,
    "min_nip_version": "0.4",
    "required_scopes": ["agent:invoke"]
  }
}
```

### 2.4 请求流程：ActionFrame → TaskFrame

```
Consumer                    Anchor Node                     NOP Orchestrator
  │                              │                                │
  │── ActionFrame ──────────→   │                                │
  │   (action_id, params)       │                                │
  │                              │── 验证 NID + scope             │
  │                              │── 查找 action → DAG 模板       │
  │                              │── 构建 TaskFrame ─────────→   │
  │                              │     (DAG, context, budget)     │
  │                              │                                │── DelegateFrame → Workers
  │                              │                                │── SyncFrame 等待
  │                              │   ←── AlignStream(结果) ───── │
  │  ←── CapsFrame(结果) ────── │                                │
```

---

## 2A. Bridge Node（NWP 扩展）

> 由 [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md) 在 v1.0-alpha.3 引入，是 Gateway Node 拆分的**第二**半。Anchor Node 守在 NPS 集群的前门（入站 NPS → 路由 NPS），Bridge Node 守在*出向*边界：NPS 帧**出**到非-NPS 系统。

### 2A.1 定义

**Bridge Node** 是把 NPS 帧与非-NPS 协议互相翻译的 NWP 节点类型。每次请求无状态，不参与集群拓扑。

| 属性 | 值 |
|------|------|
| 节点类型 | `bridge` |
| NWM node_type | `"bridge"` |
| NDP node_kind | `"bridge"`（或包含 `"bridge"` 的数组形式）|
| NDP bridge_protocols | 支持的目标数组，如 `["http", "grpc"]` |
| 帧入口 | 携带 `bridge_target` 的 ActionFrame (0x11) |

> **方向说明。** Bridge Node 承担 **NPS → 外部** 方向。LabAcacia 历史上发布在 `compat/*-bridge` 下的包（MCP / A2A / gRPC）方向相反（外部 → NPS），已重命名为 `compat/*-ingress`，把名称冲突让出。详见 [NPS-CR-0001 §3.2](../cr/NPS-CR-0001-anchor-bridge-split.md) 与 CR-0001 §6。

### 2A.2 Bridge Node 职责

| 职责 | 对应协议 | 描述 |
|------|---------|------|
| **NPS 帧入站** | NWP | 接受携带 `bridge_target`（外部端点 + 协议选择）的 ActionFrame |
| **出向翻译** | HTTP / gRPC / MCP / A2A | 把 NPS payload 编码为目标协议的请求格式 |
| **响应翻译** | NWP | 把目标方响应解码为 `CapsFrame` 返回 |
| **认证转接** *(可选)* | NIP / vendor | 在可映射时把 NIP 凭据转发为目标协议的认证 token，否则用 Bridge 实例侧配置的 vendor 凭据 |
| **可观测性** | NOP Context | 在 span 上标注目标协议 + 端点，让端到端追踪能跨边界 |

### 2A.3 v1.0-alpha.3 标准目标

下表是参考 Bridge Node 实现期望支持的最小集；按目标协议的具体 `bridge_target` schema 推迟到后续 CR。

| 目标 | NDP `bridge_protocols` 取值 | 说明 |
|------|--------------------------|------|
| HTTP / HTTPS | `"http"` | REST 与 streaming |
| gRPC | `"grpc"` | unary 与 streaming |
| Model Context Protocol | `"mcp"` | LLM 工具集成目标 |
| Agent-to-Agent | `"a2a"` | Google A2A v0.2 |

通过未来 CR 加入新 `bridge_protocols` 取值即可注册更多协议。

### 2A.4 已移除：Gateway Node

> **Gateway Node**（v1.0-alpha.3 移除）—— 原 "Gateway Node" 角色作为 **Anchor Node**（本节 §2）继承下来。引入 **Bridge Node** 这个新名是因为 "bridge" 一词被 `compat/*-bridge` 历史占用；那些包已重命名为 `compat/*-ingress`。完整背景见 NPS-CR-0001。

---

## 3. Vector Proxy Layer（向量化中间层）

### 3.1 核心思路

在传统数据库（RDS/NoSQL）前加一层向量代理，实现：

```
AI Agent ←→ Vector Proxy Layer ←→ 传统数据库
           (向量空间)              (SQL/文档)
```

- **写入路径**：结构化数据 → 生成嵌入向量 → 存入向量索引 + 原始数据写入 DB
- **查询路径**：Agent 语义查询 → 向量相似度检索 → 返回压缩的向量表示 + 关键字段
- **效果**：AI 不再需要解析完整 SQL 结果集，直接消费向量化摘要，token 消耗大幅下降

### 3.2 架构

```
                    NWP Memory Node 接口
                          │
               ┌──────────┴──────────┐
               │  Vector Proxy Layer  │
               │                      │
               │  ┌────────────────┐  │
               │  │ Embedding Engine│  │  ← 嵌入模型（本地/远程）
               │  └───────┬────────┘  │
               │          │           │
               │  ┌───────┴────────┐  │
               │  │ Vector Index   │  │  ← ANN 索引（HNSW/IVF）
               │  │ (内存/磁盘)    │  │
               │  └───────┬────────┘  │
               │          │           │
               │  ┌───────┴────────┐  │
               │  │ Schema Mapper  │  │  ← DB schema ↔ AnchorFrame 映射
               │  └────────────────┘  │
               └──────────┬───────────┘
                          │
               ┌──────────┴──────────┐
               │   传统数据库         │
               │  (PostgreSQL/MySQL/  │
               │   MongoDB/...)       │
               └─────────────────────┘
```

### 3.3 工作模式

#### 3.3.1 Schema 映射（启动时）

1. 扫描目标数据库 schema（表、列、类型、关系）
2. 自动生成 NWP AnchorFrame schema
3. 对文本类列生成嵌入向量，建立 ANN 索引
4. 注册为 NWP Memory Node（含 `vector_search` capability）

#### 3.3.2 查询（运行时）

| 查询方式 | 适用场景 | token 节约 |
|----------|---------|-----------|
| **向量语义查询** | "找与 X 相似的记录" | ~70-80%（只返回 top-K 摘要向量） |
| **结构化查询 + 向量摘要** | 精确过滤 + AI 可读摘要 | ~40-60%（过滤后向量化压缩） |
| **透传查询** | 需要完整原始数据 | 0%（fallback 到传统模式） |

#### 3.3.3 响应格式

```json
{
  "frame": "0x04",
  "anchor_ref": "sha256:...",
  "count": 5,
  "data": [
    {
      "_id": "product:1001",
      "_score": 0.95,
      "_embedding": [0.12, -0.34, ...],
      "name": "Widget Pro",
      "price": 29.99
    }
  ],
  "vector_meta": {
    "model": "text-embedding-3-small",
    "dimensions": 256,
    "index_type": "hnsw"
  },
  "token_est": 85
}
```

对比传统方式返回完整行数据（可能含数十列无关字段），向量模式只返回
top-K 相似结果 + 嵌入向量 + 关键字段，token_est 从数百降至两位数。

### 3.4 一致性策略

Vector Proxy Layer 在数据写入/更新时存在向量索引与原始数据之间的一致性窗口。
实现者 MUST 选择并声明以下一致性模式之一：

| 模式 | 机制 | 适用场景 | 一致性延迟 |
|------|------|---------|-----------|
| **最终一致（默认）** | WAL/CDC 监听 → 异步更新向量索引 | 读多写少、可接受短暂脏读 | 通常 < 1s |
| **强一致** | 双写事务（DB 写入 + 向量索引更新在同一事务中） | 金融、实时决策 | 同步，无延迟 |

**AnchorFrame TTL 与 Schema 变更**：当数据库 Schema 发生变更（加列/删列）时：

1. Vector Proxy MUST 主动失效受影响的 AnchorFrame（不等待 TTL 过期）
2. 重新扫描 Schema → 生成新 AnchorFrame → 增量重建向量索引
3. 向量索引重建期间，MUST 回退到透传查询模式（fallback passthrough）

### 3.5 与 NWP QueryFrame 的集成

Vector Proxy Layer 对上层完全透明，消费方使用标准 NWP QueryFrame：

```json
{
  "frame": "0x10",
  "anchor_ref": "sha256:...",
  "vector_search": {
    "field": "_embedding",
    "vector": [0.11, -0.33, ...],
    "top_k": 5,
    "metric": "cosine"
  },
  "fields": ["name", "price", "_score"]
}
```

NWP v0.5 已支持 `vector_search` 字段（§6.4），Vector Proxy Layer 只需实现
该接口即可无缝对接。

---

## 4. AaaS 合规性要求

### 4.1 合规层级

| 层级 | 名称 | 要求 |
|------|------|------|
| **Level 1** | Basic | Anchor Node + NIP 认证 + NWM 服务目录 |
| **Level 2** | Standard | Level 1 + NOP 编排 + OpenTelemetry 追踪 + Token Budget |
| **Level 3** | Advanced | Level 2 + Vector Proxy Layer + K-of-N 容错 + 审计日志 |

### 4.2 Level 1 — Basic 合规

| 要求 ID | 描述 | 对应协议 |
|---------|------|---------|
| L1-01 | MUST 部署 Anchor Node 作为唯一服务入口 | NWP |
| L1-02 | MUST 通过 NIP 验证消费方 NID | NIP |
| L1-03 | MUST 发布 NWM manifest 含所有可用 Action | NWP |
| L1-04 | MUST 支持 NPS 统一端口 17433 | NCP |
| L1-05 | MUST 返回 NPS 标准状态码和错误帧 | NCP |
| L1-06 | SHOULD 支持 HTTP 模式和原生模式双模承载 | NCP |

### 4.3 Level 2 — Standard 合规

| 要求 ID | 描述 | 对应协议 |
|---------|------|---------|
| L2-01 | MUST 使用 NOP TaskFrame 进行内部任务编排 | NOP |
| L2-02 | MUST 在 TaskFrame.context 中注入 OpenTelemetry trace | NOP |
| L2-03 | MUST 支持 NPT Token Budget，响应中包含 token_est | NPT |
| L2-04 | MUST 支持 NOP 预检（preflight）机制 | NOP |
| L2-05 | MUST 实现 NOP 重试和超时语义 | NOP |
| L2-06 | SHOULD 支持异步 Action（ActionFrame.async=true） | NWP |
| L2-07 | SHOULD 实现 AlignStream 背压控制 | NOP |
| L2-08 | 维护成员注册表的 Anchor Node MUST 实现 `topology.snapshot` 与 `topology.stream` 保留查询类型，详见 [NPS-2 §12](../NPS-2-NWP.cn.md)。Version 计数器 MUST 在 Anchor 生命周期内单调，进程重启时 SHOULD 通过 rebase + `anchor_state.version_rebased` 事件保持订阅一致性。 | NWP §12 |

### 4.4 Level 3 — Advanced 合规

| 要求 ID | 描述 | 对应协议 |
|---------|------|---------|
| L3-01 | MUST 部署 Vector Proxy Layer 支持向量化查询 | NWP |
| L3-02 | MUST 支持 NWP vector_search 接口 | NWP §6.4 |
| L3-03 | MUST 实现 K-of-N 同步容错（SyncFrame.min_required） | NOP |
| L3-04 | MUST 记录审计日志（NOP §8.3）| NOP |
| L3-05 | MUST 实现 scope 委托链安全（最大 3 层）| NIP + NOP |
| L3-06 | SHOULD 支持 Schema 自动发现（DB schema → AnchorFrame） | NWP |
| L3-07 | SHOULD 支持向量索引热更新（数据变更时增量重建） | Vector Proxy |

---

## 5. 端到端场景示例

### 场景：消费方 Agent 请求数据分析服务

```
Consumer Agent                Anchor Node               NOP              Vector Memory Node    Action Worker
     │                             │                     │                      │                   │
     │── ActionFrame ────────→    │                     │                      │                   │
     │   action: analysis.run     │                     │                      │                   │
     │   params: {query: "..."}   │                     │                      │                   │
     │                             │── verify NID ──→   │                      │                   │
     │                             │── build DAG ──→    │                      │                   │
     │                             │   TaskFrame         │                      │                   │
     │                             │   ┌─────────┐      │                      │                   │
     │                             │   │ fetch    │──→   │── DelegateFrame ──→ │                   │
     │                             │   │ analyze  │      │                      │── QueryFrame ──→ DB
     │                             │   │ summarize│      │                      │  (vector_search)  │
     │                             │   └─────────┘      │  ←── CapsFrame ───── │  (top-K, 85 NPT) │
     │                             │                     │                      │                   │
     │                             │                     │── DelegateFrame ─────────────────────→  │
     │                             │                     │                                          │
     │                             │                     │  ←── AlignStream(result) ──────────────  │
     │                             │                     │                      │                   │
     │  ←── CapsFrame ──────────  │  ←── result ────── │                      │                   │
     │   (向量化摘要, 85+50 NPT)   │                     │                      │                   │
```

传统方式：fetch 返回完整行数据 ~500 NPT → analyze 处理 ~300 NPT = **800+ NPT**
AaaS 向量化方式：fetch 返回 top-K 向量摘要 ~85 NPT → analyze ~50 NPT = **~135 NPT**

**Token 节约 ~83%**。

---

## 6. 与现有 NPS 协议的关系

| 本 Profile 组件 | 依赖的 NPS 协议 | 是否需要协议变更 |
|----------------|----------------|----------------|
| Anchor Node | NWP v0.8 | 需要：把 `node_type: "gateway"` 改名为 `"anchor"`（CR-0001）；遗留 `"gateway"` 拒绝。L2 集群注册表通过 `topology.snapshot` / `topology.stream` 暴露（CR-0002）|
| Bridge Node | NWP v0.8 | 需要：新增 `node_type: "bridge"` 取值（CR-0001）|
| 请求路由 | NOP v0.4 | 不需要：复用 TaskFrame/DelegateFrame |
| 身份认证 | NIP v0.3 | 不需要：复用 NID + scope |
| 向量查询 | NWP v0.5 §6.4 | 不需要：复用 vector_search |
| Vector Proxy | NWP + NCP | 不需要：实现层面的中间件 |
| Token 计量 | NPT v0.1 | 不需要：复用 token_est |
| 审计追踪 | NOP v0.4 §8.3 | 不需要：复用 context.trace_id |

**所需协议变更**：NWP NWM `node_type` 枚举须接受 `"anchor"` 与 `"bridge"`（替换被移除的 `"gateway"`）；NDP AnnounceFrame 新增 `node_kind` / `cluster_anchor` / `bridge_protocols` 字段。除 `"gateway"` 移除外，wire 层均 additive —— 详见 [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md)。

---

## 7. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.4 | 2026-04-27 | §4.3 新增 L2-08 要求，强制维护成员注册表的 Anchor Node 实现 `topology.snapshot` / `topology.stream`，详见 [NPS-CR-0002](../cr/NPS-CR-0002-anchor-topology-queries.md) 与 [NPS-2 §12](../NPS-2-NWP.cn.md)。§2 引言段中"留给 v1.0-alpha.4"的占位措辞替换为具体的 L2 强制声明。§2.2 集群注册表行从"可选"升级为"L1 可选、L2 强制"。§6 协议关系表对齐到 NWP v0.8 以引用新 §12 接口。Depends-On 升级：NPS-2（NWP v0.7 → v0.8）。 |
| 0.3 | 2026-04-26 | **破坏性**（按 [NPS-CR-0001](../cr/NPS-CR-0001-anchor-bridge-split.md)）。§2 Gateway Node 重命名为 **Anchor Node**，wire/manifest 示例同步更新（`node_type: "anchor"`、`nwm_version: "0.7"`、`min_nip_version: "0.4"`）；§2.2 新增可选集群注册表行。新增 §2A **Bridge Node**（NPS↔非-NPS 协议翻译、`bridge_protocols` 声明、与 `compat/*-bridge` 的方向澄清）。§1.2 / §1.3 架构图与 §6 协议关系表更新；L1 §4.2 / §5 引用改指向 Anchor Node。§6 协议变更行从"新增 gateway"改为"重命名 gateway → anchor + 新增 bridge"。Depends-On 升级为 NCP v0.6（RFC-0001）+ NWP v0.7（CR-0001）+ NIP v0.4（RFC-0003）+ NDP v0.5（CR-0001 字段）。 |
| 0.1 | 2026-04-15 | 初始草案：AaaS 架构总览、Gateway Node 定义、Vector Proxy Layer 设计、三级合规要求 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
