[English Version](./README.md) | 中文版

# NPS — Neural Protocol Suite

> **AI 时代的完整互联网基础协议族**

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-Phase%201-green.svg)]()
[![Release](https://img.shields.io/badge/release-v1.0.0--alpha.5-orange.svg)](CHANGELOG.cn.md)
[![NCP](https://img.shields.io/badge/NCP-v0.6-5b8cff.svg)]()
[![NWP](https://img.shields.io/badge/NWP-v0.10-4af0b0.svg)]()
[![NIP](https://img.shields.io/badge/NIP-v0.6-7b61ff.svg)]()
[![NDP](https://img.shields.io/badge/NDP-v0.6-f0a050.svg)]()
[![NOP](https://img.shields.io/badge/NOP-v0.4-ff8c42.svg)]()

NPS 是面向 AI Agent 和模型的完整 Web 基础协议族，由五个子协议组成，覆盖 AI 通信、Web 访问、身份认证、节点发现与多 Agent 编排。

---

## 协议族概览

```
┌──────────────────────────────────────────────────────┐
│  NOP  Neural Orchestration Protocol   多 Agent 编排  │
├──────────────────────────────────────────────────────┤
│  NDP  Neural Discovery Protocol       节点发现        │
├──────────────────────────────────────────────────────┤
│  NIP  Neural Identity Protocol        Agent 身份      │
├──────────────────────────────────────────────────────┤
│  NWP  Neural Web Protocol             Web 节点访问    │
├──────────────────────────────────────────────────────┤
│  NCP  Neural Communication Protocol   AI 间通信       │
└──────────────────────────────────────────────────────┘
```

| 协议 | 类比 | 规范版本 | 实现状态 | 端口 (默认/独立) |
|------|------|----------|----------|-----------------|
| **NCP** Neural Communication Protocol | Wire Format | v0.6 | ✅ 参考实现完成；原生模式连接前导（NPS-RFC-0001 Phase 1）helper 已落地 | 17433 / — |
| **NWP** Neural Web Protocol | HTTP | v0.10 | ✅ Memory / Action / Complex / **Anchor** / **Bridge** Node；Anchor Node `topology.snapshot` / `topology.stream`（NPS-CR-0002）；完整 NWP 错误码（30 个常量，alpha.5）| 17433 / 17434 |
| **NIP** Neural Identity Protocol | TLS / PKI | v0.6 | ✅ CA + 身份验证器；X.509 NID 证书 + ACME `agent-01`（RFC-0002）；保证等级（RFC-0003）；声誉日志 Phase 1+2+3（RFC-0004）| 17433 / 17435 |
| **NDP** Neural Discovery Protocol | DNS | v0.6 | ✅ 注册表 + 公告验证器；AnnounceFrame 新增 `activation_mode` + `node_kind`/`cluster_anchor`/`bridge_protocols`（NPS-CR-0001）| 17433 / 17436 |
| **NOP** Neural Orchestration Protocol | SMTP / MQ | v0.4 | ✅ 编排引擎 + 安全加固已实现 | 17433 / 17437 |

### 参考 daemon 部署

NPS 在生产环境跑作 **三层、六个常驻服务** —— 完整设计见
[`docs/daemons/architecture.cn.md`](docs/daemons/architecture.cn.md)，
二进制在 [`tools/daemons/`](tools/daemons/)。

| 层 | Daemon | 端口 | 状态（alpha.5）|
|----|--------|------|-----------------|
| 1（本机基础设施）| [`npsd`](tools/daemons/npsd/)             | 17433 | L1 + sub-NID 签发 + per-NID inbox 队列 |
| 1（本机基础设施）| [`nps-runner`](tools/daemons/nps-runner/) | —     | Phase 1 骨架 |
| 2（接入网关）| [`nps-gateway`](tools/daemons/nps-gateway/)   | 8080  | Phase 1 骨架 |
| 2（接入网关）| [`nps-registry`](tools/daemons/nps-registry/) | 17436 | SQLite 实仓注册表（持久化 + TTL lazy purge）|
| 3（信任锚点）| [`nps-cloud-ca`](tools/daemons/nps-cloud-ca/)  | 17435 | Deferral 骨架（指向 [`tools/nip-ca-server`](tools/nip-ca-server/)）|
| 3（信任锚点）| [`nps-ledger`](tools/daemons/nps-ledger/)      | 17440 | RFC-0004 Phase 3：SQLite + Merkle + STH + inclusion proof + STH gossip 联邦 |

---

## 为什么需要 NPS？

现有 Web 协议为人类浏览器设计，AI Agent 访问时面临三个根本问题：

- **语义解析开销**：HTML/CSS/JS 表现层对 AI 无意义，浪费大量 token
- **Schema 重复传输**：每次响应携带完整结构定义，高频访问场景浪费显著
- **无 Agent 身份概念**：无法在协议层区分 AI 访问与人类访问，无法声明 Agent 能力和权限范围

NPS 从零开始，通过 **AnchorFrame Schema 锚定**、**Cognon (CGN) 标准化计量** 和 **NID 身份体系** 重新设计 AI 互联网基础设施。

---

## 核心特性

**Token Economy 第一**
- **Schema 锚定**：首次请求后后续调用仅传 SHA-256 引用，典型场景节约 30–60% token 消耗。
- **Cognon (CGN)**：跨模型的标准化 token 计量单位，支持按预算裁剪响应（Token Budget）。

**三类神经节点**
- `Memory Node`：数据存储与检索（RDS / NoSQL / 文件 / 向量数据库）
- `Action Node`：操作与服务调用
- `Complex Node`：综合数据与处理，支持节点图谱遍历（Depth 控制）

**AI-Native 身份**
每个 Agent 持有 NID（Neural Identity Descriptor），格式为 `urn:nps:agent:{issuer}:{id}`，由 NIP CA 颁发，携带能力声明和访问 scope，节点在协议层强制校验。

**统一端口 & 传输双模**
- **统一端口**：全协议族共用 **17433**，通过帧类型码天然路由。
- **双模承载**：支持 **HTTP 模式**（Overlay 部署，防火墙友好）与 **原生模式**（高性能 TCP/QUIC）。

---

## 仓库结构

```
nps/
├── spec/                    # 语言无关规范文档（SSoT）
│   ├── NPS-0-Overview.md    # 套件总览 v0.3
│   ├── NPS-1-NCP.md         # NCP 规范 v0.5
│   ├── NPS-2-NWP.md         # NWP 规范 v0.5
│   ├── NPS-3-NIP.md         # NIP 规范 v0.3
│   ├── NPS-4-NDP.md         # NDP 规范 v0.3
│   ├── NPS-5-NOP.md         # NOP 规范 v0.4
│   ├── frame-registry.yaml  # 机器可读帧注册表 v0.9
│   ├── error-codes.md       # 统一错误码命名空间
│   ├── status-codes.md      # NPS 原生状态码 + HTTP 映射
│   ├── token-budget.md      # CGN 计量规范
│   ├── services/
│   │   └── NPS-AaaS-Profile.md  # AaaS 合规性规范 v0.2
│   └── rfcs/                # RFC 流程 + 4 份草案（NCP 前导 / X.509+ACME NID / 身份保证等级 / 声誉日志）
├── impl/
│   ├── dotnet/              # C# / .NET 10 参考实现（含 samples/ + benchmarks/）
│   ├── python/              # Python SDK v1.0.0-alpha.3（Phase 1 完成；alpha.3 suite-wide 同步）
│   ├── typescript/          # TypeScript SDK v1.0.0-alpha.3（Phase 2 完成；alpha.3 suite-wide 同步）
│   ├── java/                # Java SDK v1.0.0-alpha.3（Phase 2 完成；alpha.3 suite-wide 同步）
│   ├── rust/                # Rust SDK v1.0.0-alpha.3（Phase 2 完成；alpha.3 suite-wide 同步）
│   └── go/                  # Go SDK v1.0.0-alpha.3（Phase 2 完成；alpha.3 suite-wide 同步）
├── tools/
│   ├── daemons/                # 六个常驻服务。4 个 OSS 打 bundle 发到 labacacia/nps-daemons（npsd / nps-runner / nps-gateway / nps-registry）；2 个 cloud daemon 私有发到 innolotus/nps-cloud-ca + innolotus/nps-ledger
│   ├── nip-ca-server/          # NIP CA Server — C# / ASP.NET Core；独立发布到 labacacia/nip-ca-server（example/ 收录 5 个冻结的参考移植）
│   ├── release/                # 发布同步脚本（dev → 各独立发布仓）
│   └── mirror-to-gitee/        # Gitee 镜像同步脚本（GitHub → Gitee，labacacia URL 改写）
├── compat/
│   ├── mcp-ingress/          # MCP Ingress v1.0.0-alpha.3（LabAcacia.McpIngress）
│   ├── a2a-ingress/          # A2A Ingress v1.0.0-alpha.3（LabAcacia.A2aIngress）
│   └── grpc-ingress/         # gRPC Ingress v1.0.0-alpha.3（LabAcacia.GrpcIngress）
└── demos/                  # 同时单独发布在 github.com/labacacia/NPS-examples
    ├── nps-demo/            # 端到端业务 demo —— NIP 身份 → AnchorFrame → NOP → DiffFrame
    ├── nwp-graph-walk/      # NWP Complex Node §11 —— depth 扇出 + X-NWP-Trace 环路检测
    ├── ingress-playground/   # 一个 NWP Action Node 被 MCP + A2A + gRPC 同时前置
    └── cross-sdk-interop/   # 四语言客户端（dotnet/python/node/go）对同一个 Memory Node 做 diff
```

> **单独展示仓库。** Tier-1 三个演示（`nwp-graph-walk`、`ingress-playground`、
> `cross-sdk-interop`）同时作为一个精选仓库发布在
> [`labacacia/NPS-examples`](https://github.com/labacacia/NPS-examples)
> （[Gitee 镜像](https://gitee.com/labacacia/NPS-examples)）。代码的唯一
> 源仍然在这里；单独仓库的意义是"可发现性"，并且每个演示都有按
> 原理 / 作用 / 演示了什么 / 运行结果 组织的 README。

---

## 实现状态

### C# / .NET（`impl/dotnet/`）

| 组件 | 版本 | 状态 | 内容 |
|------|------|------|------|
| `NPS.Core` | 1.0.0-alpha.5 | ✅ 可用 | 帧编解码（MsgPack/JSON）、双头模式（4B/8B）、帧注册表、Anchor 缓存 |
| `NPS.NWP` | 1.0.0-alpha.5 | ✅ 可用 | Memory / Action / Complex / Anchor / Bridge Node 中间件；`/.nwm`·`/.schema`·`/actions`·`/invoke`·`/query`·`/system.task.*`；图谱遍历 + X-NWP-Depth + 环路检测；SSRF + 幂等 + priority + 异步任务生命周期；**30 个 NWP wire 错误码常量**（NwpErrorCodes）；`NPS-SERVER-UNSUPPORTED` 状态码 |
| `NPS.NIP` | 1.0.0-alpha.5 | ✅ 可用 | CA 库（密钥生成、证书签发/吊销、OCSP、CRL）、`NipIdentVerifier` 6 步身份验证；gossip 错误码（`REPUTATION_GOSSIP_FORK` / `REPUTATION_GOSSIP_SIG_INVALID`）；`AssuranceLevel` 空字符串修复 |
| `NPS.NDP` | 1.0.0-alpha.5 | ✅ 可用 | NDP 帧类型（Announce/Resolve/Graph）、内存注册表（TTL 淘汰）、公告签名验证器 |
| `NPS.NOP` | 1.0.0-alpha.5 | ✅ 可用 | DAG 编排引擎（条件求值、输入映射、K-of-N 同步、重试/退避）+ §8.2 委托链深度限制 + §8.4 callback SSRF 防护及指数退避重试 |
| `tools/nip-ca-server` | 1.0.0-alpha.5 | ✅ 可用 | NIP CA Server —— C# / ASP.NET Core 10、PostgreSQL、Docker。独立发布到 [`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server)（唯一打 release 的实现）；5 个其它语言参考移植（Python / TypeScript / Java / Rust / Go）冻结在 `1.0.0-alpha.2`，放在 `tools/nip-ca-server/example/` 下。|
| Compat 接入 | 1.0.0-alpha.5 | ✅ 可用 | MCP Ingress（JSON-RPC 2.0，MCP 2024-11-05）、A2A Ingress（Google A2A v0.2）、gRPC Ingress（HTTP/2，4 个 unary RPC）；由 NPS-CR-0001 从 `*-bridge` 重命名 —— 详见 `docs/compat/index.md` |
| Daemons | 1.0.0-alpha.5 | ✅ 可用 | `npsd`（L1 + sub-NID 签发 + per-NID inbox 队列）、`nps-runner`（Phase 1 骨架）、`nps-gateway`（Phase 1 骨架）、`nps-registry`（SQLite 持久化 + TTL lazy purge）、`nps-ledger`（RFC-0004 Phase 3：SQLite + Merkle + STH + inclusion proof + STH gossip 联邦）；详见 [`docs/daemons/architecture.cn.md`](docs/daemons/architecture.cn.md) |
| Samples | — | ✅ 可用 | `samples/NPS.Samples.NopDag` —— 真 HTTP 的 3 节点 NOP DAG 端到端；`demos/nps-demo` —— 4 幕业务 demo（NIP → AnchorFrame → NOP → DiffFrame）|
| Benchmarks | — | ✅ 可用 | `benchmarks/NPS.Benchmarks.TokenSavings` → **相对 REST 节省 45.0% CGN**（超过 Phase 1 ≥30% 出口）；`benchmarks/NPS.Benchmarks.WireSize` → **MsgPack 相对 JSON 减少 63.6%**（超过 Phase 2 ≤50% 出口）|

.NET 解决方案测试总量：**655 tests**（NPS.Core / NWP / NIP（含 AssuranceLevel + Reputation + gossip）/ NDP / NOP / Anchor / Bridge / Ncp.NcpPreamble / samples / benchmarks），加上 **48 ingress tests**（15 mcp + 18 a2a + 15 grpc）。

### Python（`impl/python/`）

| 组件 | 版本 | 状态 | 内容 |
|------|------|------|------|
| `nps-lib` | 1.0.0-alpha.5 | ✅ 可用 | NCP + NWP + NIP + NDP + NOP 全协议实现，asyncio + httpx，Ed25519 签名，211 测试，97% 覆盖率。`nps_sdk.nwp.error_codes`（30 个 NWP 错误码常量）；`AssuranceLevel.from_wire("")` 返回 ANONYMOUS 修复。Python 导入模块仍为 `nps_sdk`。 |

### TypeScript（`impl/typescript/`）

| 组件 | 版本 | 状态 | 内容 |
|------|------|------|------|
| `@labacacia/nps-sdk` | 1.0.0-alpha.5 | ✅ 可用 | NCP + NWP + NIP + NDP + NOP 全协议实现，ESM + CJS 双输出，Ed25519 签名，284 测试，98.75% 覆盖率；`NwpErrorCodes`（30 个常量）；`NpsStatusCodes.NPS_SERVER_UNSUPPORTED`；`fromWire("")` 返回 ANONYMOUS 修复 |

### Java（`impl/java/`）

| 组件 | 版本 | 状态 | 内容 |
|------|------|------|------|
| `nps-java` | 1.0.0-alpha.5 | ✅ 可用 | NCP + NWP + NIP + NDP + NOP 全协议实现，Java 21，MsgPack + JSON 双编码，Ed25519 内置签名，AES-256-GCM 密钥加密，112 测试；`NwpErrorCodes` 类（30 个常量）；`fromWire("")` 返回 ANONYMOUS 修复 |

### Rust（`impl/rust/`）

| 组件 | 版本 | 状态 | 内容 |
|------|------|------|------|
| `nps-rs` | 1.0.0-alpha.5 | ✅ 可用 | NCP + NWP + NIP + NDP + NOP 全协议实现，Rust stable，MsgPack + JSON 双编码，Ed25519 签名，AES-256-GCM 密钥加密，Tokio 异步，109 测试；`nps_nwp::error_codes` 模块（30 个常量）；workspace 版本统一 |

### Go（`impl/go/`）

| 组件 | 版本 | 状态 | 内容 |
|------|------|------|------|
| `github.com/labacacia/NPS-sdk-go` | 1.0.0-alpha.5 | ✅ 可用 | NCP + NWP + NIP + NDP + NOP 全协议实现，Go 1.25+，MsgPack + JSON 双编码，Ed25519 内置签名，AES-256-GCM 密钥加密，96 测试；`nwp/error_codes.go`（30 个 ErrAuth* / ErrQuery* / … 常量） |

---

## 快速开始（C#）

### 1. 注册 NCP 核心与 NWP 服务
```csharp
services.AddNpsCore(opt => {
    opt.DefaultTier = EncodingTier.MsgPack;
});

services.AddNwp(opt => {
    opt.DefaultTokenBudget = 1000;
});
```

### 2. 构建并编码 NWP 查询帧
```csharp
var query = new QueryFrame {
    AnchorRef = "sha256:a3f9b2c1...",
    Filter = JsonSerializer.SerializeToElement(new { status = "active" }),
    Limit = 50
};

byte[] wire = codec.Encode(query); // 自动处理 4-byte/8-byte 帧头
```

---

## 路线图

**Phase 0 — 规范统一（已完成）**
- [x] 全部 5 份子协议规范文档 v0.2+ draft
- [x] `frame-registry.yaml` 机器可读帧注册表 v0.2
- [x] `error-codes.md` / `status-codes.md` 错误码与 HTTP 映射
- [x] `token-budget.md` CGN 计量规范
- [x] `services/NPS-AaaS-Profile.md` AaaS 合规性规范 v0.1

**Phase 1 — 核心构建（2026 Q3）— ✅ 已交付**
- [x] `NPS.Core` C# 参考实现 — 帧编解码库
- [x] `NPS.NWP` C# 参考实现 — Memory / Action / Complex / Anchor / Bridge Node 中间件
- [x] `NPS.NIP` C# 参考实现 — CA 库 + 身份验证器（OCSP / CRL）
- [x] `NPS.NDP` C# 参考实现 — 帧类型 + 注册表 + 公告验证器
- [x] `NPS.NOP` C# 参考实现 — 编排引擎全量实现 + 安全加固
- [x] NIP CA Server OSS v0.1 —— **6 个语言变体**（C# / Python / TypeScript / Java / Rust / Go）
- [x] Python SDK `nps-lib` v1.0.0-alpha.5（NCP + NWP + NIP + NDP + NOP，211 测试；`nps_sdk.nwp.error_codes`；import 模块 `nps_sdk`）
- [x] Token-savings 基准 → 相对 REST 聚合节省 45.0%（达成 Phase 1 ≥30% 出口标准）
- [x] **NPS-RFC-0001** Accepted（NCP 连接前导）—— 六种 SDK 全覆盖
- [x] **NPS-RFC-0003** Accepted（Agent 身份保证等级）—— `AssuranceLevel` 空字符串修复跨六 SDK 交付
- [x] **NPS-RFC-0004** Accepted（NID 声誉日志）—— Phase 1+2+3：SQLite + Merkle + STH + inclusion proof + STH gossip 联邦
- [x] `npsd` L1 daemon：sub-NID 签发 + per-NID inbox 队列
- [ ] NuGet 包正式发布（`NPS.Core`、`NPS.NWP` 等）
- [ ] PyPI 正式发布（`nps-lib`）

**Phase 2 — 生态扩展（2026 Q4）— 🚧 进行中**
- [x] TypeScript SDK v1.0.0-alpha.5（@labacacia/nps-sdk：NCP + NWP + NIP + NDP + NOP，ESM + CJS，Ed25519，284 tests；`NwpErrorCodes`；`NPS_SERVER_UNSUPPORTED`）
- [x] Java SDK v1.0.0-alpha.5（nps-java：NCP + NWP + NIP + NDP + NOP，Java 21，Ed25519，112 tests；`NwpErrorCodes`）
- [x] Rust SDK v1.0.0-alpha.5（nps-rs：NCP + NWP + NIP + NDP + NOP，Rust stable，Ed25519，109 tests；`nps_nwp::error_codes`）
- [x] Go SDK v1.0.0-alpha.5（github.com/labacacia/NPS-sdk-go：NCP + NWP + NIP + NDP + NOP，Go 1.25+，Ed25519，96 tests；`nwp/error_codes.go`）
- [x] **MCP Ingress** v1.0.0-alpha.5（`LabAcacia.McpIngress`，MCP 2024-11-05，JSON-RPC 2.0；NPS-CR-0001 由 `MCP Bridge` 重命名）
- [x] **A2A Ingress** v1.0.0-alpha.5（`LabAcacia.A2aIngress`，Google A2A v0.2；NPS-CR-0001 由 `A2A Bridge` 重命名）
- [x] **gRPC Ingress** v1.0.0-alpha.5（`LabAcacia.GrpcIngress`，4 个 unary RPC；NPS-CR-0001 由 `gRPC Bridge` 重命名）
- [x] **6 个 daemon 二进制** 在 `tools/daemons/`（`npsd` / `nps-runner` / `nps-gateway` / `nps-registry` / `nps-cloud-ca` / `nps-ledger`）—— 详见 [`docs/daemons/architecture.cn.md`](docs/daemons/architecture.cn.md)
- [x] **NPS-CR-0001** Implemented（Anchor + Bridge Node 拆分；`compat/*-bridge` → `compat/*-ingress` 重命名）
- [x] **NPS-CR-0002** Implemented（Anchor Node 上的 `topology.snapshot` / `topology.stream` 保留查询类型；.NET 参考实现 + L2 合规套件）
- [x] **30 个 NWP wire 错误码常量** 跨全六 SDK 发布（alpha.5）
- [x] `nps-registry` SQLite 持久化注册表（持久化 + TTL lazy purge）
- [x] `nps-ledger` RFC-0004 Phase 3：STH gossip 联邦（`GET /v1/log/gossip/sth`）
- [x] 3 节点 NOP DAG 样例（`samples/NPS.Samples.NopDag`）—— 达成 Phase 2 DAG 出口标准
- [x] Wire-size 基准 → MsgPack 相对 JSON 减少 63.6%（达成 Phase 2 ≤50% 出口标准）
- [x] NOP Orchestrator 执行多节点 DAG；Claude Desktop 通过 `mcp-ingress` 集成验证
- [x] `NDP.ResolveFrame` DNS TXT 解析（`nwp://` → 物理端点）—— 全六 SDK 交付（`resolve_via_dns` / `resolveWithDns` / `ResolveViaDns`；可注入 `DnsTxtLookup`）
- [ ] **NPS Studio**（帧流可视化调试器）+ **NPS Probe**（合规检查 CLI）
- [ ] NPS Cloud 托管服务
- [ ] 各协议规范升至 Proposed / Stable 状态

---

## 文档索引

### 概念指南

| 主题 | 说明 |
|------|------|
| [帧模型](docs/concepts/frame-model.md) | NCP 帧结构、编码 Tier、Flags 位图、Schema 锚定机制 |
| [NID 身份体系](docs/concepts/nid-identity.md) | NID 格式、证书结构、六步验证、OCSP、Scope 通配符 |
| [DAG 编排](docs/concepts/dag-orchestration.md) | 执行流程、输入映射、K-of-N 同步、回调与聚合策略 |
| [节点类型](docs/concepts/node-types.md) | Memory / Action / Complex / Anchor / Bridge 节点对比 |

### .NET SDK 参考

| 包 | 说明 |
|----|------|
| [.NET SDK 索引](docs/sdk/dotnet/index.md) | 环境要求、包依赖关系、DI 注册总览 |
| [NPS.Core](docs/sdk/dotnet/nps-core.md) | 帧类型、编解码器、AnchorCache、异常体系 |
| [NPS.NWP](docs/sdk/dotnet/nps-nwp.md) | QueryFrame、Filter DSL、MemoryNodeMiddleware、NWM 清单 |
| [NPS.NIP](docs/sdk/dotnet/nps-nip.md) | NipIdentVerifier 六步验证、NipCaService、NipSigner、CA HTTP 路由 |
| [NPS.NDP](docs/sdk/dotnet/nps-ndp.md) | AnnounceFrame、INdpRegistry、NdpAnnounceValidator |
| [NPS.NOP](docs/sdk/dotnet/nps-nop.md) | TaskFrame/DAG 模型、NopOrchestrator、回调校验器、条件/输入映射 |

### 兼容性桥

| 主题 | 说明 |
|------|------|
| [桥层总览](docs/compat/index.md) | MCP / A2A / gRPC 何时选哪个；共同设计与非目标 |
| [MCP Bridge 详解](docs/compat/mcp-ingress.md) | 1:N 上游模型、tool 名编码、异步生命周期、header 语义 |
| [A2A Bridge 详解](docs/compat/a2a-ingress.md) | 1:1 AgentCard 映射、skill 查找、任务状态转换、内存绑定 |
| [gRPC Bridge 详解](docs/compat/grpc-ingress.md) | bytes 透传原理、双错误映射、多语言 client |

### 基准报告

| 报告 | 结果 |
|------|------|
| [REST vs NWP token 节省](docs/benchmarks/token-savings.md) | 聚合 **45.0%** CGN 节省（S1 43.1% / S2 44.0% / S3 54.2%）—— 超过 Phase 1 ≥30% 出口标准 |
| [Tier-1 JSON vs Tier-2 MsgPack wire size](docs/benchmarks/wire-size.md) | 稳态帧聚合字节减少 **63.6%** —— 超过 Phase 2 ≤50% 出口标准 |

### 协议规范

| 文档 | 说明 |
|------|------|
| [NPS 总览](spec/NPS-0-Overview.md) | 套件入口，帧命名空间一览 |
| [NCP 规范](spec/NPS-1-NCP.md) | 帧线格式、编码 Tier、双头模式 |
| [NWP 规范](spec/NPS-2-NWP.md) | Web 节点协议、Filter DSL、流式响应 |
| [NIP 规范](spec/NPS-3-NIP.md) | 身份协议、Ed25519 签名、CRL/OCSP |
| [NDP 规范](spec/NPS-4-NDP.md) | 节点发现、TTL 广播、图谱同步 |
| [NOP 规范](spec/NPS-5-NOP.md) | DAG 编排、委托链、K-of-N |
| [Cognon (CGN) 计量](spec/token-budget.md) | 跨模型标准化 Token 计量单位 |
| [错误码命名空间](spec/error-codes.md) | 全协议统一错误码 |
| [状态码映射表](spec/status-codes.md) | NPS 原生状态码与 HTTP 映射 |
| [AaaS 合规性规范](spec/services/NPS-AaaS-Profile.md) | Anchor Node + Bridge Node、Vector Proxy、L1/L2/L3 合规级别 |
| [Daemon 架构](docs/daemons/architecture.cn.md) | 六-daemon、三层参考部署拓扑 |
| [RFC 流程 + 草案](spec/rfcs/README.md) | RFC-0001 NCP 前导 · 0002 X.509+ACME NID · 0003 身份保证等级 · 0004 声誉日志 |

---

## 归属

| 产出 | 归属 |
|------|------|
| NPS 规范文档 | LabAcacia / INNO LOTUS PTY LTD |
| 参考实现 (OSS) | LabAcacia |
| NPS Cloud 服务 | INNO LOTUS PTY LTD |

**LabAcacia** 是 INNO LOTUS PTY LTD 旗下的开源实验室。Apache 2.0 授权。

---

## License

[Apache 2.0](LICENSE) © 2026 INNO LOTUS PTY LTD
