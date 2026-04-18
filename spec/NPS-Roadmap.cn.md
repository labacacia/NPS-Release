[English Version](./NPS-Roadmap.md) | 中文版

# NPS 路线图

**Version**: 0.2  
**Date**: 2026-04-12  
**归属**: LabAcacia / INNO LOTUS PTY LTD  

---

## 节奏策略

每个 Phase 分三段：

```
① Spec Sprint（2 周）   — 本 Phase 所有规范冻结
② Impl Sprint（6–8 周） — 实现、测试、文档
③ Review Gate（1 周）   — 社区/内部评审，决定是否进入下一 Phase
```

## 版本约定

| 版本号 | 含义 |
|--------|------|
| `v0.x-draft` | 内部草稿，可破坏性变更 |
| `v0.x-alpha` | 公开预览，API 不稳定 |
| `v0.x-beta` | 功能完整，欢迎外部测试 |
| `v1.0` | 规范冻结，生产可用 |

---

## Phase 0 — 规范统一（2026 Q2）

**目标**：建立 NPS 完整规范骨架，统一帧空间和命名，输出可供社区评论的 v0.1-draft。

- [x] NPS-0-Overview.cn.md v0.2 完成
- [x] NPS-1-NCP.cn.md v0.3-draft 完成（传输双模、可配帧大小、ErrorFrame）
- [x] NPS-2-NWP.cn.md v0.2-draft 完成（AnchorFrame Node 发布、NPT）
- [x] NPS-3-NIP.cn.md v0.2-draft 完成（metadata 字段、NPS 状态码）
- [x] NPS-4-NDP.cn.md v0.2-draft 完成（统一端口、NPS 状态码）
- [x] NPS-5-NOP.cn.md v0.2-draft 完成（统一端口、NPS 状态码）
- [x] frame-registry.yaml v0.2 完成（含 ErrorFrame 0xFE）
- [x] error-codes.cn.md v0.2 完成（含 NPS 状态码映射）
- [x] status-codes.cn.md v0.1 完成（NPS 原生状态码 + HTTP 映射）
- [x] token-budget.cn.md v0.1 完成（NPT 计量 + tokenizer 解析链）
- [ ] LabAcacia 仓库公开，Discussions 开启
- [ ] v0.2-draft GitHub Release 发布

**完成标准**：所有子协议文档无空节，frame-registry.yaml 涵盖全部帧（含 ErrorFrame），status-codes.cn.md 覆盖所有错误码映射，GitHub 仓库公开。

---

## Phase 1 — 核心实现（2026 Q3）

**目标**：NCP + NWP + NIP 三层生产可用，NIP CA Server OSS v0.1，C# + Python SDK 初版。

- [ ] NPS.Core NuGet（.NET 10，帧编解码 Tier-1/2，AnchorFrame 缓存，ErrorFrame，EXT 帧头）
- [ ] NPS.NWP：Memory Node + Action Node（SQL Server / PostgreSQL / Redis）
- [ ] NPS.NWP：NWM 清单自动生成，Overlay 模式中间件，NPT token 计量
- [ ] NPS.NIP：IdentFrame / TrustFrame / RevokeFrame 编解码
- [ ] NIP CA Server OSS v0.1（Docker，SQLite，REST API + CLI）
- [ ] C# SDK NuGet 包发布（v0.1.0-alpha，.NET 10）
- [ ] Python SDK 初版 PyPI（v0.1.0a）

**完成标准**：
- NPS.Core 单元测试覆盖率 ≥ 90%
- Memory Node QueryFrame 往返集成测试通过
- NIP CA Server Docker Compose 一键启动文档可用
- Token 节约基准：单会话对比 REST ≥ 30%

---

## Phase 2 — 完整协议族（2026 Q4）

**目标**：NDP + NOP 实现，Complex Node，MCP/A2A 适配器，TypeScript SDK，Tier-2 生产就绪。

- [ ] NPS.NDP：AnnounceFrame / ResolveFrame / GraphFrame（DNS TXT + 本地 Multicast）
- [ ] NPS.NOP：TaskFrame / DelegateFrame / SyncFrame / AlignStream
- [ ] Complex Node 完整实现（跨源聚合，子节点路由）
- [ ] MCP Bridge（NWP Memory Node → MCP Resource）
- [ ] A2A Bridge（NOP TaskFrame → A2A Task）
- [ ] TypeScript SDK npm 包（v0.1.0-beta）
- [ ] Tier-2 MsgPack 生产验证（体积 ≤ JSON 的 50%）

**完成标准**：
- NDP ResolveFrame 可解析 nwp:// 到物理端点（DNS TXT 模式）
- NOP Orchestrator 可执行 3 节点 DAG 任务
- Claude Desktop 通过 MCP Bridge 访问 NWP Memory Node

---

## Phase 3 — 生态验证（2027 Q1–Q2）

**目标**：真实场景 PoC，NPS Cloud CA v1.0 上线，建立事实标准基础。

- [ ] NPS Cloud CA v1.0（多区域 HA，实时 OCSP，¥299/月 Professional Plan）
- [ ] LangChain / AutoGen / CrewAI 集成适配包
- [ ] FinTech PoC（Open Banking 场景，跨组织 TrustFrame）
- [ ] 车联网 PoC（设备 NID，StreamFrame 实时遥测）
- [ ] Token 节约基准测试报告（公开发布）
- [ ] NIP CA Server OSS v1.0（PostgreSQL + Web Admin UI）
- [ ] GitHub Stars ≥ 500

---

## Phase 4 — 标准化（2027 Q3 起）

**目标**：推动 NPS 成为 W3C / IETF 正式标准，NPS 1.0 规范冻结。

- [ ] 多厂商联合支持声明（≥ 3 家）
- [ ] W3C WebAI Community Group 提案
- [ ] IETF Internet-Draft（NCP + NWP 核心规范）
- [ ] NPS 1.0 规范冻结
- [ ] ISO/IEC JTC 1 评估
- [ ] Tier-3 MatrixTensor 规范

---

## 里程碑依赖图

```
Phase 0                Phase 1                  Phase 2             Phase 3
──────                 ────────                 ────────            ────────
[规范骨架]
    │
    ├──→ [NPS.Core] ──→ [NWP Memory/Action] ──→ [Complex Node]
    │         │                  │              [MCP Bridge] ──→ [框架集成]
    │    [NIP CA OSS] ──────────────────────→  [A2A Bridge]
    │         │
    └──→ [Python SDK] ──────────────────────→  [TS SDK]
                                               [NDP]
                                               [NOP]          [Cloud CA]──→ [PoC]
```

---

## 风险登记册

| ID | 风险 | 概率 | 影响 | 缓解 |
|----|------|------|------|------|
| R01 | 规范变更导致实现返工 | 高 | 高 | Phase 0 规范先冻结再实现；变更走 RFC 流程 |
| R02 | MCP 生态快速演进，兼容层失效 | 中 | 中 | mcp-bridge 独立版本化 |
| R03 | Token 节约效果不及预期（<30%）| 中 | 高 | Phase 1 即测基准；AnchorFrame 命中率是关键 |
| R04 | NIP CA 私钥安全事件 | 低 | 极高 | HSM 接口预留；年度密钥轮换强制执行 |
| R05 | 竞品先达到类似定位 | 中 | 中 | NPS 差异在 Token Economy；加速 OSS 发布 |
| R06 | Phase 3 PoC 合作方资源不到位 | 中 | 中 | 备选：内部 Demo 数据集替代真实合作方 |
| R07 | W3C/IETF 标准化周期过长 | 高 | 低 | 事实标准路径（GitHub 社区采用）优先于正式 RFC |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
