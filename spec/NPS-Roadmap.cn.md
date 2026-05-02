[English Version](./NPS-Roadmap.md) | 中文版

# NPS 路线图

**Version**: 0.4  
**Date**: 2026-04-30  
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

## Phase 0 — 规范统一（2026 Q2）— ✅ 完成

**目标**：建立 NPS 完整规范骨架，统一帧空间和命名，输出可供社区评论的 v0.2-draft。

- [x] `NPS-0-Overview.md` v0.3
- [x] `NPS-1-NCP.md` v0.6（传输双模、可配帧大小、ErrorFrame）
- [x] `NPS-2-NWP.md` v0.8（AnchorFrame Node 发布、CGN、拓扑查询）
- [x] `NPS-3-NIP.md` v0.5（metadata 字段、NPS 状态码、X.509 NID 原型）
- [x] `NPS-4-NDP.md` v0.5
- [x] `NPS-5-NOP.md` v0.4
- [x] `frame-registry.yaml` v0.9（含 ErrorFrame 0xFE）
- [x] `error-codes.md` v1.0（含 NPS 状态码映射、NIP 证书错误码、NWP 拓扑错误码）
- [x] `status-codes.md` v0.2（NPS 原生状态码 + HTTP 映射）
- [x] `token-budget.md` v0.2（CGN 计量 + tokenizer 解析链）
- [x] `services/NPS-AaaS-Profile.md` v0.4（Anchor/Bridge Node、VPL、L1/L2/L3、NPS-CR-0002）
- [x] `services/NPS-Node-Profile.md` v0.1（L1/L2/L3 + 激活模式）
- [x] `services/conformance/NPS-Node-L1.md` v0.1（21 个 TC-N1-* 用例）
- [x] `services/conformance/NPS-Node-L2.md` v0.1（10 个 TC-N2-* 用例，拓扑查询）
- [x] LabAcacia 仓库公开，Discussions 开启；`NPS-Dev` monorepo 按设计保持私有
- [x] 已发布：alpha.1（2026-04-10）、alpha.2（2026-04-19）、alpha.3（2026-04-26）、alpha.4（2026-04-30）、alpha.5（2026-05-01）

---

## Phase 1 — 核心实现（2026 Q3）— ✅ 已交付

**目标**：NCP + NWP + NIP + NDP + NOP 在六种参考语言中生产可用；NIP CA Server 六语言参考部署。

### SDK

| 语言 | 包名 | 状态 |
|------|------|------|
| .NET       | `LabAcacia.NPS.Core` + `.NWP` + `.NWP.Anchor` + `.NWP.Bridge` + `.NIP` + `.NDP` + `.NOP` | ✅ v1.0.0-alpha.5（655 tests）|
| Python     | `nps-lib`（PyPI）                  | ✅ v1.0.0-alpha.5（211+ tests，≥97% coverage）|
| TypeScript | `@labacacia/nps-sdk`（npm）        | ✅ v1.0.0-alpha.5（284+ tests）|
| Java       | `com.labacacia.nps:nps-java`（Maven Central）| ✅ v1.0.0-alpha.5（112+ tests）|
| Rust       | `nps-sdk` + 6 个同生态 crate（crates.io）| ✅ v1.0.0-alpha.4（109 tests）|
| Go         | `github.com/labacacia/NPS-sdk-go`  | ✅ v1.0.0-alpha.4（96 tests）|

### NIP CA Server（六语言参考部署）

| 语言 | 技术栈 | 状态 |
|------|--------|------|
| C# / .NET  | ASP.NET Core + SQLite + Docker  | ✅ v0.1 |
| Python     | FastAPI + SQLite + Docker       | ✅ v0.1 |
| TypeScript | Fastify + SQLite + Docker       | ✅ v0.1 |
| Java       | Spring Boot 3.4 + SQLite        | ✅ v0.1 |
| Rust       | Axum + SQLite + Docker          | ✅ v0.1 |
| Go         | net/http stdlib + SQLite        | ✅ v0.1 |

### .NET 服务器运行时（参考实现）

- [x] `NPS.Core` — 帧编解码、AnchorCache、EXT 帧头
- [x] `NPS.NWP` — Memory Node 中间件（SQL Server / PostgreSQL），284 integration tests
- [x] `NPS.NWP.Anchor` — `IAnchorTopologyService` + `topology.snapshot` / `topology.stream`（NPS-CR-0002）
- [x] `NPS.NOP` — DAG 校验器 + 编排引擎，委托链深度限制，SSRF 防护，指数退避重试，429 tests
- [x] `NPS.NIP` — CA 库：密钥生成、证书签发 / 吊销、OCSP、CRL

### Phase 1 交付的 RFC / CR

- [x] **NPS-RFC-0001** — NCP 连接前导码 `b"NPS/1.0\n"`（Accepted，全六 SDK）
- [x] **NPS-RFC-0002 Phase A/B** — X.509 NID + ACME `agent-01` 原型（Draft，全六 SDK；IANA PEN 待申请）
- [x] **NPS-RFC-0003** — Agent 身份保证级别 `anonymous`/`attested`/`verified`（Accepted，全六 SDK）
- [x] **NPS-RFC-0004 Phase 1+2** — NID 声誉日志（CT 风格）；SQLite + RFC 9162 Merkle 树 + operator-signed STH + 包含证明（`nps-ledger`）
- [x] **NPS-CR-0001** — Anchor/Bridge Node 拆分；`NWP.Gateway` 退役；`compat/*-ingress` 重命名
- [x] **NPS-CR-0002** — Anchor Node 拓扑查询；L2 合规测试套件（10 tests）

### 常驻 Daemon

| Daemon         | alpha.5 状态 |
|----------------|-------------|
| `npsd`         | ✅ L1 + 子 NID 签发 + 每 NID 收件箱队列（17 integration tests）|
| `nps-registry` | ✅ SQLite 持久化注册中心（SqliteNdpRegistry，10 tests）|
| `nps-ledger`   | ✅ Phase 3：SQLite + Merkle + STH + 包含证明 + STH gossip（33 tests）|
| `nps-runner`   | Phase 1 骨架（L3 运行时推迟）|
| `nps-gateway`  | Phase 1 骨架（Internet 入站网关推迟）|
| `nps-cloud-ca` | 存根（2027 Q1+）|

### 完成标准

- [x] 六语言 SDK 单元覆盖率 ≥ 90%
- [x] Memory Node `QueryFrame` 往返集成测试通过
- [x] NIP CA Server Docker Compose 一键启动文档可用
- [x] Token 节约基准 ≥ 30%（实测 45.0%）
- [x] 电报体积基准 vs JSON（实测聚合减少 63.6%）

---

## Phase 2 — 生态扩展（2026 Q4）— 🚧 进行中

**目标**：现有生态（MCP、A2A、gRPC）适配器，更丰富的 SDK 示例，Tier-2 MsgPack 生产硬化。

- [x] `compat/mcp-ingress/` — NWP Memory/Action/Complex Node ↔ MCP 2024-11-05 适配器（`LabAcacia.McpIngress` v1.0.0-alpha.4）
- [x] `compat/a2a-ingress/` — NOP `TaskFrame` ↔ A2A Task 适配器（`LabAcacia.A2aIngress` v1.0.0-alpha.4）
- [x] `compat/grpc-ingress/` — NWP Memory/Action/Complex Node ↔ gRPC 适配器（`LabAcacia.GrpcIngress` v1.0.0-alpha.4）
- [x] Tier-2 MsgPack 线路体积基准（聚合较 JSON 减少 63.6%）
- [x] Token 节约基准（聚合较 REST 减少 45.0% CGN）
- [x] NOP Orchestrator 端到端执行 3 节点 DAG
- [x] Claude Desktop 通过 `mcp-ingress` 访问 NWP Memory Node
- [ ] `NDP.ResolveFrame` 通过 DNS TXT 解析 `nwp://` 到物理端点
- [ ] 首个参考产品：**NPS Studio**（人类可视化调试器）+ **NPS Probe**（Agent Coder 合规检查 CLI）

---

## alpha.5 发布 — 2026-05-01 ✅

### alpha.5 已完成

| 事项 | 备注 |
|------|------|
| **NPS-RFC-0004 Phase 3** — `nps-ledger` STH gossip | `GossipState` + `GossipService` + `GET /v1/log/gossip/sth`；13 新测试 |
| **AaaS-Profile L2-09** — 默认 `reputation_policy` | SHOULD 要求；定义推荐最小策略 |
| **`NWP-RESERVED-TYPE-UNSUPPORTED`** 在 AnchorNodeMiddleware | HTTP 501；规范新增 `NPS-SERVER-UNSUPPORTED` 状态码 |
| **`topology:read` 能力门控** | `AnchorNodeOptions.RequireTopologyCapability`；`X-NWP-Capabilities` 头 |
| **`cgn_est` 每事件字段** | `TopologyEventEnvelope.CgnEst` = UTF-8/4 估算 |
| **AssuranceLevel `from_wire("")` 修复** | Python / TypeScript / Java SDK |
| **规范 CN 文档同步** | `error-codes.cn.md`、`RFC-0004.cn.md`、`status-codes.cn.md` 均已更新 |

### 推迟到 alpha.6

| 事项 | 备注 |
|------|------|
| **NPS-CR-0002 Phase 2** — 服务端 Anchor 中间件推送拓扑更新 | alpha.5 交付查询 API；push/notify 推迟；需新 CR 定义授权模型 |
| 非 .NET SDK 移植 NPS-CR-0002 `AnchorNodeClient` 拓扑客户端 | .NET 参考已完成；Python/TS/Go/Java/Rust 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0004 声誉助手（`ReputationLogClient`）| .NET 参考已完成；全六 SDK 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0003 保证级别执行助手 | .NET 已接入；其他 SDK 只有枚举，无执行逻辑 |
| **NPS-RFC-0002** 晋级 Draft → Proposed/Accepted | 阻塞于 IANA PEN 分配（`nid-assurance-level` 当前用临时 OID `1.3.6.1.4.1.99999`）|

## alpha.6 任务队列

v1.0.0-alpha.6 待开展任务（未启动）：

### 进行中的 RFC / CR

| 事项 | 备注 |
|------|------|
| **NPS-CR-0002 Phase 2** — 服务端 Anchor 中间件推送拓扑更新 | alpha.5 交付查询 API；push/notify 推迟 |
| **NPS-RFC-0002** 晋级 Draft → Proposed/Accepted | 阻塞于 IANA PEN 分配 |

### SDK 功能缺口

| 事项 | 备注 |
|------|------|
| 非 .NET SDK 移植 NPS-CR-0002 `AnchorNodeClient` 拓扑客户端 | .NET 参考已完成；Python/TS/Go/Java/Rust 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0004 声誉助手（`ReputationLogClient`）| .NET 参考已完成；全六 SDK 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0003 保证级别执行助手 | .NET 已接入；其他 SDK 只有枚举，无执行逻辑 |

### 协议 / 规范事项

| 事项 | 备注 |
|------|------|
| `NDP.ResolveFrame` DNS TXT 解析（`nwp://` → 物理端点）| 已规范化；所有 SDK 尚未实现 |
| `nps-gateway` L2 Internet 入站网关（`:8080`→`:443` TLS 终止，NCP over TLS）| alpha.5 仅骨架；L2 合规推迟 |
| `nps-runner` L3 FaaS 任务运行时 | 仅骨架；完整实现在 Phase 3 范围 |

### 工具链

| 事项 | 备注 |
|------|------|
| **NPS Studio** — NPS 帧流可视化调试器 | Phase 2 目标；尚未开始 |
| **NPS Probe** — Agent Coder 合规检查 CLI | Phase 2 目标；尚未开始 |

---

## Phase 3 — 生态验证（2027 Q1–Q2）

**目标**：真实场景 PoC，NPS Cloud CA v1.0 上线，建立事实标准基础。

- [ ] NPS Cloud CA v1.0（多区域 HA，实时 OCSP，Professional Plan）
- [ ] LangChain / AutoGen / CrewAI 集成适配包
- [ ] FinTech PoC（Open Banking 场景，跨组织 `TrustFrame`）
- [ ] 车联网 PoC（设备 NID，`StreamFrame` 实时遥测）
- [ ] Token 节约基准测试报告（公开发布）
- [ ] NIP CA Server OSS v1.0（PostgreSQL + Web Admin UI）
- [ ] NIP CA Server 以 NWP Memory Node 为后端自托管（dogfooding）
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
    │         │                  │              [mcp-ingress] ──→ [框架集成]
    │    [NIP CA OSS] ──────────────────────→  [a2a-ingress]
    │         │                               [grpc-ingress]
    └──→ [六语言 SDK] ────────────────────────→ [SDK parity]
                                               [NDP DNS TXT]
                                               [NOP 编排]  [Cloud CA]──→ [PoC]
```

---

## 风险登记册

| ID | 风险 | 概率 | 影响 | 缓解 |
|----|------|------|------|------|
| R01 | 规范变更导致实现返工 | 高 | 高 | Phase 0 规范先冻结再实现；变更走 RFC 流程 |
| R02 | MCP 生态快速演进，ingress 适配层失效 | 中 | 中 | mcp-ingress 独立版本化 |
| R03 | Token 节约效果不及预期（<30%）| 中 | 高 | Phase 1 即测基准（实测 45%）；AnchorFrame 命中率是关键 |
| R04 | NIP CA 私钥安全事件 | 低 | 极高 | HSM 接口预留；年度密钥轮换强制执行 |
| R05 | 竞品先达到类似定位 | 中 | 中 | NPS 差异在 Token Economy；加速 OSS 发布 |
| R06 | Phase 3 PoC 合作方资源不到位 | 中 | 中 | 备选：内部 Demo 数据集替代真实合作方 |
| R07 | W3C/IETF 标准化周期过长 | 高 | 低 | 事实标准路径（GitHub 社区采用）优先于正式 RFC |
| R08 | IANA PEN 分配延迟 | 中 | 低 | RFC-0002 使用临时 OID 发布；IANA PEN 不阻塞 alpha 发布 |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
