[English Version](./NPS-Roadmap.md) | 中文版

# NPS 路线图

**Version**: 0.8
**Date**: 2026-08-01
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

- [x] `NPS-0-Overview.md` v0.4
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
| .NET       | `LabAcacia.NPS.Core` + `.NWP` + `.NWP.Anchor` + `.NWP.Bridge` + `.NIP` + `.NDP` + `.NOP` | ✅ v1.0.0-alpha.11（655 tests）|
| Python     | `nps-lib`（PyPI）                  | ✅ v1.0.0-alpha.11（211+ tests，≥97% coverage）|
| TypeScript | `@labacacia/nps-sdk`（npm）        | ✅ v1.0.0-alpha.11（284+ tests）|
| Java       | `com.labacacia.nps:nps-java`（Maven Central）| ✅ v1.0.0-alpha.11（112+ tests）|
| Rust       | `nps-sdk` + 6 个同生态 crate（crates.io）| ✅ v1.0.0-alpha.11（109 tests）|
| Go         | `github.com/labacacia/NPS-sdk-go`  | ✅ v1.0.0-alpha.11（96 tests）|

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
| `nps-ingress`  | Phase 1 骨架（Internet 入站网关推迟）|
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

- [x] `compat/mcp-ingress/` — NWP Memory/Action/Complex Node ↔ MCP 2024-11-05 适配器（`LabAcacia.McpIngress` v1.0.0-alpha.11）
- [x] `compat/a2a-ingress/` — NOP `TaskFrame` ↔ A2A Task 适配器（`LabAcacia.A2aIngress` v1.0.0-alpha.11）
- [x] `compat/grpc-ingress/` — NWP Memory/Action/Complex Node ↔ gRPC 适配器（`LabAcacia.GrpcIngress` v1.0.0-alpha.11）
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
| **NPS-CR-0002 Phase 2** — 服务端 Anchor 中间件推送拓扑更新 | .NET 参考实现已通过 `AnchorNodeMiddleware` + `IAnchorTopologyService` 落地 push/notify；非 .NET 移植仍见下方 |
| 非 .NET SDK 移植 NPS-CR-0002 `AnchorNodeClient` 拓扑客户端 | .NET 参考已完成；Python/TS/Go/Java/Rust 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0004 声誉助手（`ReputationLogClient`）| .NET 参考已完成；全六 SDK 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0003 保证级别执行助手 | .NET 已接入；其他 SDK 只有枚举，无执行逻辑 |
| **NPS-RFC-0002** 晋级 Draft → Proposed/Accepted | 已由 NPS-CR-0004（2026-05-08）关闭：IANA PEN **65715** 已分配；OID arc `1.3.6.1.4.1.65715` 替换临时 `1.3.6.1.4.1.99999`；RFC-0002 晋级 Draft → Proposed（wire-in 落地于 alpha.6）|

## alpha.6 任务队列

v1.0.0-alpha.11 待开展任务：

### 进行中的 RFC / CR

| 事项 | 备注 |
|------|------|
| **NPS-CR-0002 Phase 2** — 服务端 Anchor 中间件推送拓扑更新 | .NET 参考实现完成；alpha.6 关闭 `node_kind` 兼容窗口，要求 `topology.filter.node_roles` |
| **NPS-RFC-0002** 晋级 Draft → Proposed/Accepted | 阻塞于 IANA PEN 分配 |

### SDK 功能缺口

| 事项 | 备注 |
|------|------|
| 非 .NET SDK 移植 NPS-CR-0002 `AnchorNodeClient` 拓扑客户端 | .NET 参考已完成；Python/TS/Go/Java/Rust 待移植 |
| 非 .NET SDK 移植 NPS-RFC-0004 声誉助手（`ReputationLogClient`）| .NET 仅有 Phase 1 数据类型，无客户端；全六 SDK 需完整客户端 |
| ~~非 .NET SDK 移植 NPS-RFC-0003 保证级别执行助手~~ | ✅ 已完成 —— 全六 SDK 均有完整 `AssuranceLevel` 枚举 + 执行逻辑 |

### 协议 / 规范事项

| 事项 | 备注 |
|------|------|
| `NDP.ResolveFrame` DNS TXT 解析（`nwp://` → 物理端点）| 已规范化；所有 SDK 尚未实现 |
| `nps-ingress` L2 Internet 入站网关（`:8080`→`:443` TLS 终止，NCP over TLS）| alpha.5 仅骨架；L2 合规推迟 |
| `nps-runner` L3 FaaS 任务运行时 | 仅骨架；完整实现在 Phase 3 范围 |

### 工具链

| 事项 | 备注 |
|------|------|
| **NPS Studio** — NPS 帧流可视化调试器 | Phase 2 目标；尚未开始 |
| **NPS Probe** — Agent Coder 合规检查 CLI | Phase 2 目标；尚未开始 |

---

## alpha.7 任务队列

v1.0.0-alpha.7 待开展任务：

### SDK 功能缺口（alpha.6 遗留 —— 发布硬性门槛）

每次 SDK 发布必须保证六种语言在同一功能水位上。以下条目从 alpha.6 延续，
必须在 alpha.7 打标签前全部完成。

| 事项 | 范围 | 备注 |
|------|------|------|
| NPS-CR-0002 `AnchorNodeClient` | ✅ 完成（2026-05-17） | 全五 SDK 非 .NET 端口：`get_snapshot` + `subscribe`（各语言对应 stream/async-generator/channel）+ 拓扑数据类型（MemberInfo、TopologySnapshot、TopologyFilter、TopologyEvent × 5）|
| NPS-RFC-0004 `ReputationLogClient` | 全六 SDK（含 .NET）| .NET 仅有 Phase 1 数据类型；需完整客户端（Phase 2 Merkle / STH / 包含证明）覆盖所有 SDK |

### 新规范 / 实现

| 事项 | 状态 | 备注 |
|------|------|------|
| **NPS-CR-0005** — NIP CA RA 注册授权模型 | ✅ 完成（2026-05-17） | .NET 参考实现：`EnrollmentTier` 枚举、`Ca/Ra/` 策略 + 存储层、4 个 enrollment 端点、4 个新错误码；`db/003_ra_model.sql` PostgreSQL 迁移脚本 |
| **#51 CGN Profile 换算规范** | ✅ 完成（2026-05-17） | `cgn-profiles.yaml` 新增 Google Gemini、Meta Llama、Mistral 系列；`token-budget.md` §2.3 同步更新 |
| **NWP + NOP OpenTelemetry 埋点** | ✅ 完成（2026-05-17） | NPS-sdk-dotnet 新增 `ActivitySource` + `System.Diagnostics.Metrics`；关闭 NPS-sdk-dotnet#5 |

### 进行中的 CR / RFC

| 事项 | 状态 | 备注 |
|------|------|------|
| **NPS-RFC-0002** 晋级 Proposed → Accepted | ✅ 完成（2026-05-17） | OQ-3 已决议（延后至后续 RFC）；无剩余未解 OQ |

---

## alpha.13 — 2026-06-13 ✅（对等与边缘；跨 alpha.13–15 交付完毕）

> **主题**：*对等与边缘（Parity & Edge）* —— 让六个参考 SDK 达到真正的**功能**对等（而非仅源码存在）、推进全部五个协议规范、并立起 L2/L3 守护进程边缘。
>
> **结果（在 alpha.13–15 跨版本全部交付）**：六 SDK 功能对等；五个协议规范全部推进（NCP v0.9 / NWP v0.14 / NIP v0.10 / NDP v0.9 / NOP v0.7）；L2/L3 守护进程边缘 —— `nps-ingress` NCP-over-TLS 终结器（`NcpTlsListener`、ALPN `nps/1.0`、mTLS、`NCP-NID-MISMATCH` session-NID 绑定）与 `nps-runner` L3（CR-0007 租约 + 续约、`SpawnSpec`、worker 生命周期）。
>
> 详细实施计划见 [`docs/roadmap.md`](../docs/roadmap.md)。

**发布闸**（打标签前全部必须交付）：

1. **SDK 功能对等**（硬闸）—— 补齐 `SDK_ALIGNMENT_ALPHA11` 的缺口：把完整的 **Anchor/Bridge Node**、**CGN / token-budget**、**reputation-policy** 实现从 .NET 参考移植到 Python / TypeScript / Java / Rust / Go。源码存在不再充分；每种语言都必须通过等价于 .NET 的 Anchor/Bridge + CGN + reputation 测试套件。
2. **五协议推进**（发布规则）—— 每个协议都要有实质的规范 + SDK 内容：
   - **NCP v0.8** —— 将 **RFC-0006**（原生模式传输绑定）由 Draft → Proposed；原生模式 TLS 绑定（ALPN `nps/1.0`、双向 TLS）、会话恢复票据。与 `nps-ingress` L2 耦合。
   - **NWP v0.14** —— Bridge Node 正式合规章节 + `bridge_target` 往返测试向量（对等驱动）。
   - **NIP v0.10** —— 边缘 mTLS 短时/可续期证书 profile；关联 `nps-ingress` 证书处理。
   - **NDP v0.9** —— AnnounceFrame liveness/health 字段 + 解析期失效检查。
   - **NOP v0.7** —— **NPS-CR-0007**（NOP ↔ L3 运行时集成）：任务认领协议、`spawn_spec_ref` 语义、idle/max-runtime 上报。与 `nps-runner` L3 耦合。
3. **Daemon L2/L3** —— `nps-ingress` L2（NCP over TLS、ALPN `nps/1.0`、双向 TLS、`:8080`→`:443` 终结、L2 合规 TC-N2-*）+ `nps-runner` L3 FaaS 运行时（NOP 任务调度、经 `spawn_spec_ref` 的 worker 生命周期、同步屏障协调）。
4. **C++/PHP 降级** —— 明确移出"官方完整 SDK 集"；作为*计划中*跟踪，不阻塞（见 Phase 1 SDK 表下方注释）。

**逐协议交付**（具体帧 / 字段 / 错误码）：

| 事项 | 备注 |
|------|------|
| **NCP v0.9** | Tier-3 BinaryVector v1（`binary_vector.v1`、`NPBV` Payload、MessagePack 元数据 + little-endian float32 向量段）用于 NWP 向量搜索；RFC-0006 原生模式 TLS 绑定（ALPN `nps/1.0`、mTLS、session-NID 绑定 `NCP-NID-MISMATCH`、恢复票据；§7.5）；NopFrame (0x07) 保活/心跳帧（null 载荷，双向）；`HelloFrame.ping_interval_ms`（uint32，0=禁用）；`NCP-KEEPALIVE-TIMEOUT`（`NPS-SERVER-TIMEOUT`）；§7.6 死节点检测（3 × ping_interval_ms）；2^32 帧或 24h 前重新密钥；`NCP-REKEY-REQUIRED` |
| **NWP v0.14** | Bridge Node 正式合规（§16）+ `bridge_target` 往返向量；NWM `manifest_version` 改为 uint32 单调递增计数器；新增 `manifest_updated_at`（ISO 8601）；所有 `GET /.nwm` 响应 MUST 返回 `X-NWM-Version`；条件请求 `If-None-Match: <uint32>` |
| **NIP v0.10** | §6.1 短时/可续期边缘 mTLS 证书 profile；`IdentFrame.node_roles`（array[string]）Phase 1–2 自声明；Phase 3 经 `id-nps-node-roles` 扩展（65715.2.2）CA 证明；`NIP-CERT-NODE-ROLES-MISMATCH` |
| **NDP v0.9** | `AnnounceFrame` liveness 字段 `health` / `last_seen` + §3.2.1 解析期失效 `NDP-RESOLVE-STALE`；`heartbeat_interval_ms`（uint32，默认 60000）+ 公告期 `NDP-ANNOUNCE-STALE`；`spawn_spec_ref`（字符串引用）解析为 SpawnSpec，正式模式 §3.1.2（oci_image / command / resource_limits：cpu_millicores/memory_mb，Profile L3）；§9 联邦转发环路检测 |
| **NOP v0.7** | **NPS-CR-0007** NOP ↔ L3 运行时（§8：任务认领租约、`NOP-CLAIM-CONFLICT`、`NOP-SPAWN-SPEC-INVALID`、`NOP-RUNTIME-IDLE-TIMEOUT`、`NOP-RUNTIME-MAX-RUNTIME`；合规 `TC-N3-*`）；`TaskFrame.result_ttl_seconds`（uint32，默认 3600）、`NOP-TASK-RESULT-EXPIRED`；`NOP-STREAM-NAK-UNRESOLVABLE`（被驱逐帧的 NAK）；frame-registry NopFrame 0x07 注册为 stable |

---

## alpha.14 — 2026-06-26 ✅

| 事项 | 备注 |
|------|------|
| **NCP Tier-3 BinaryVector —— SDK 实现** | `binary_vector.v1` 编解码落到六个 SDK 源码树 + 畸形帧/客户端错误合规覆盖（规范在 alpha.13 落地） |
| **入站 NWP Bridge 服务端适配器** | `McpInboundServer` / `A2aInboundServer` / `GrpcInboundService` + host `BridgeServerHandler` / `BridgeServerApp` —— 外部 MCP/A2A/gRPC 客户端调用被 Bridge front 的 NPS node；默认安全（NID + verifier、请求体上限、派发超时、错误信息脱敏） |
| **原生模式 NWP 服务端** | `NwpNativeNodeServer` 在 `NcpSession` 上直接服务 `QueryFrame` / `ActionFrame` |
| **强类型远程 NIP CA 客户端** | `NipCaClient`（发现、CRL、注册/续期/吊销/验证、RFC-0002 X.509）；`/v1/crl` 新增 `issued_at` + 分离式 CA 签名 |
| **Daemon 可观测性 + 合规测试框架** | 传输中立的 `HealthProbeRenderer`；`LabAcacia.NPS.Conformance`（Node L1/L2 用例目录） |

---

## alpha.15 — 2026-06-28 ✅

> **主题**：*一致性（Consistency）* —— 跨 SDK 的 wire 正确性。
>
> **编号说明**：`1.0.0-alpha.15` 这个 CHANGELOG 标题在打标签后仍在累积内容 —— 归在它下面的 LLM/Thinking Profile 系列（NWP v0.15–v0.17、NIP v0.11）日期是 2026-07-04/05，**从未**以 alpha.15 上过任何 registry，实际以 **alpha.16** 发布（见下节）。下表三行才是 alpha.15 真正发布的内容。

| 事项 | 备注 |
|------|------|
| **NIP TrustFrame/RevokeFrame 签名载荷重对齐**（破坏性） | 签名载荷对齐当前 NPS-3 字段（`issued_at`、`serial`、`signer_nid`、`target_nid`）；`NIP-CERT-REVOKED` 命名。alpha.14 时代产生的旧签名帧不再通过验证 |
| **NDP AnnounceFrame 签名规范形式 —— 规范化且跨 SDK 一致**（破坏性） | 签名体排除 `signature`/`health`/`last_seen`/`frame`；null 可选字段省略；`heartbeat_interval_ms` 仅在缺省时取默认 `60000`，显式 `0` 按字面签名。六个 SDK 全部对齐 |
| **NDP graph + NIP revoke 守卫强制执行** | GraphFrame 256 节点/1024 边界限（`NDP-GRAPH-TOO-LARGE`）、3 跳联邦环路（`NDP-FEDERATION-LOOP`）、RevokeFrame `parent_nid`↔`parent_revoked` 规则 |

---

## alpha.16 — 2026-07-23 ✅

> **主题**：*LLM / Thinking Profile* —— 让模型服务型 Node 在 NWM 中成为一等公民，并补齐 HTTP overlay 的错误码注册表。
>
> **这才是 alpha.16 实际发布的内容** —— 不是本路线图早先版本挂在 alpha.16 标题下的 *HA & Hardening*，那部分已移至 下面的 **alpha.17** 一节。
>
> **为什么是这个号**：alpha.15 的包号在公共 registry 上已被占用，因此这批归在 `1.0.0-alpha.15` CHANGELOG 标题下的内容改以 alpha.16 重新发布。
>
> **已发布协议集**：NCP v0.9 / NWP v0.17 / NIP v0.11 / NDP v0.9 / NOP v0.7；`error-codes.md` v1.6。

| 事项 | 备注 |
|------|------|
| **NWP v0.15 —— `llm.complete` ActionFrame 契约** | 新增 §7.5：强类型请求/响应 DTO 形状、`stop_reason` 枚举、tool call 字段命名、同步/异步/流式响应语义、ErrorFrame 与 payload error 的区分规则、snake_case 键策略。无新帧类型、无新错误码 |
| **NWP v0.16 —— NWM `profiles` + LLM/Thinking Profile** | 新增 §4.2a `profiles.llm`，面向模型服务型 Action/Complex Node；"Thinking Node" 是产品侧别名，**不是**新的 `node_type`；粗粒度发现走 NIP/NDP `llm:*` capability，详细的模型/流式/工具/隐私/推理披露元数据放在 NWM。附 .NET DTO 与 helper |
| **NIP v0.11 —— `llm:*` capability 注册** | `llm:complete`、`llm:stream`、`llm:tool_call`、`llm:embed`、`llm:rerank`；TrustFrame `trust_scope` 可覆盖这些能力。无新帧字段、无新错误码 |
| **NWP v0.17 —— HTTP 绑定拒绝错误码** | 新增 §9.5 + `error-codes.md` v1.6：`NWP-HTTP-ORIGIN-FORBIDDEN`、`NWP-HTTP-CONTENT-TYPE-UNSUPPORTED`、`NWP-HTTP-ACCEPT-UNSATISFIABLE`、`NWP-HTTP-REQUEST-ID-MISMATCH`、`NWP-HTTP-FRAME-BODY-MALFORMED`、`NWP-CAPABILITY-ADVERTISED-UNIMPLEMENTED`（已声明但未实现的灰度窗口）。关闭 NPS-Dev#84 |
| **NIP CA —— RA store 持久化** | 三级 RA enrollment store（NPS-CR-0005）改为落到 CA 存储后端，不再仅内存 |
| **Bridge schema 修复 + daemon 测试隔离** | `bridge_target` 载荷契约修正；`nps-ingress` / `nps-runner` 发行构建不再把测试源码编进应用工程（内部覆盖率通过显式 test-assembly 可见性保留） |

**alpha.16 已发布的已知缺陷**（`main` 上已修，随 alpha.17 发布）：已发布的 **Go / Rust / TypeScript / Java** SDK 的 NOP `DelegateFrame` 发的是 `task_id` / `target_nid`，而 NPS-5 要求 `parent_task_id` / `target_agent_nid` —— 已对各 SDK 的 `v1.0.0-alpha.16` 标签逐一核对。Rust 另有 `SyncFrame.subtask_ids`（规范：`wait_for`）与 `AlignStream.sync_id` / `source_nid`（规范：`stream_id` / `sender_nid`）的偏差。.NET 与 Python 是正确的，因此 **alpha.16 下 .NET/Python 节点与 Go/Rust/TS/Java 节点之间的 NOP 委托无法互通**。

---

## alpha.17 — 2026-08-03 ✅ —— **已发布至全部 registry**

> **主题**：*HA & Hardening + 双向 Bridge + 可移植 Server/Conformance Profile* —— 集群在 Anchor 失联后仍存活（[NPS-CR-0009](cr/NPS-CR-0009-multi-anchor-ha.md)）、Bridge Node 成为双向协议边界（[NPS-CR-0010](cr/NPS-CR-0010-bridge-bidirectional.md)）、NIP 在 `v1.0.0-beta.1` flag day 之前拿到 Phase-3 强制开关、修复 alpha.16 发布出去的跨 SDK NOP wire-key 缺陷，并且每个协议都获得一套**可移植 server / conformance profile**，让六个 SDK 以完全一致的方式实现彼此的服务端面。
>
> **本次发布如何建成 —— 两条并行线的合并。** alpha.17 在两个 workspace 并行开发，最终 reconcile 成一条候选。一条线交付 CR-0009 多 Anchor HA、CR-0010 双向 Bridge（合并版 `bridge_inbound` 服务端，**替换**了早先重复的 `McpServerBridge` / `A2aServerBridge`）、NIP Phase-3、`TC-N2-HA-*` 向量、`nps-registry` 集群解析、CR-0009 CN 正文；另一条线交付跨六 SDK 的**可移植 server / conformance profile 波**（NWP §16.5、NCP §2.6.2、NDP registry conformance + `graph_seq`、NOP orchestrator profile、NIP §7.6）。并集在六语言全绿。
>
> **重编号**：CR-0009/0010 边缘线先重排到已发布的 alpha.16 号段之上，成为 NWP 0.18/0.19、NIP 0.12、NDP 0.10/0.11、NCP 0.10、NOP 0.8。可移植 profile 波随后在每个协议上**各再 +1**：NCP **0.11**、NWP **0.20**、NIP **0.13**、NDP **0.12**、NOP **0.9**；`error-codes.md` **1.8**、`frame-registry.yaml` 0.14。任何 "0.18-0.19 / 0.12 / 0.10-0.11 / 0.10 / 0.8 / 1.7" 的旧号是 profile 波之前的编号，已过期。

**发布闸**：

1. **规范内容** —— ✅ **已完成**。五协议推进：NCP v0.11 / NWP v0.20 / NIP v0.13 / NDP v0.12 / NOP v0.9；`error-codes.md` v1.8；`frame-registry.yaml` v0.14。
2. **跨 SDK 实现对等**（硬闸 —— 既定发布规则是*规范**与**SDK*）—— ✅ **已完成**。CR-0009 的 `cluster_epoch` / failover、NIP 的 `phase3_enforcement`、CR-0010 的入站服务器（`McpInboundServer` / `A2aInboundServer` / `GrpcInboundService`）、以及可移植 server / conformance profile，均已在六个 SDK（Go / Java / Python / Rust / TypeScript / .NET）实现，各自测试通过。
3. **NOP wire-key 修复进到*已发布*的包**（硬闸）—— ✅ **已完成**。`DelegateFrame`（`parent_task_id` / `target_agent_nid`）、`SyncFrame`（`wait_for`）、`AlignStream`（`stream_id` / `sender_nid`）的 wire key 在所有 SDK 对齐 NPS-5；TS/Java 保留旧键解码回退。
4. **合规** —— ✅ **已完成**。`TC-N2-BridgeIn-01..06` 与 `TC-N2-HA-01..09`（EN/CN 双语齐全）；`nps-registry` 已实现最高 epoch 解析 + `NDP-CLUSTER-SPLIT`，HA 族有参考对象可测。
5. **分发** —— ✅ **已完成**。Dev 已同步到 `NPS-Release/spec`、六个 standalone SDK 仓、daemon/tool 分发仓、文档面与镜像；`NPS-Release/version.yaml` 在最后一步完成 bump，alpha.17 已 tag 并发布。
6. **CN 翻译** —— ✅ CR-0009 / CR-0010 正文与 NIP §7.5 均已译；剩余 profile 波正文记在 `version-matrix.yaml` 的 `translation_lag`。
7. **状态卫生** —— ✅ **RFC-0006** Accepted（NCP v0.10 起原生模式为规范性）；**CR-0008 / CR-0009 / CR-0010** → Implemented；`spec/rfcs/README.md` + `spec/cr/README.md` 表已刷新；`CLAUDE.md` 已更新。

**逐协议交付**（均在发布候选上）：

| 协议 | 版本 | 备注 |
|---|---|---|
| **NCP** | v0.10 + v0.11 | **0.10**：RFC-0006 Accepted（原生模式传输成为规范性）；**Anchor failover 期间会话续接**（CR-0009）—— 所有权转移后连接断开 / `NCP-NID-MISMATCH` 时经 NDP §9（最高 `cluster_epoch`）或 NWP `anchor_failover` 的 `successor_nid` 重新解析。**0.11**：§2.6.2 **Native Server 互操作 profile** —— 认证先于 preamble 的顺序、有界 preamble/Hello 读取、分配安全的 Hello 上限、准入前失败静默处理 |
| **NWP** | v0.18 + v0.19 + v0.20 | **0.18（CR-0009）**：定稿 `anchor_failover`（`successor_nid` / `cluster_epoch` / `reason`）与 `anchor_quorum_lost`（`quorum_size` / `available`）；`cluster_epoch`（uint64）所有权栅栏；`NWP-ANCHOR-NOT-LEADER`、`NWP-ANCHOR-EPOCH-FENCED`。单 Anchor 集群保持 epoch 1。**0.19（CR-0010）**：Bridge Node 双向化 —— Outbound + Inbound 两组 MUST 列表，MCP 入站 MUST 同时服务 `resources/*` 与 `tools/*`；§16 拆为出站（§16.1.1）/ 入站（§16.1.2）profile + 方向声明（§16.2）+ 错误映射表（§16.3）；`compat/*-ingress` 并入 `NPS.NWP.Bridge`；`NWP-BRIDGE-DIRECTION-UNSUPPORTED`。**0.20**：§16.5 **可移植 Node/Bridge server profile** + 跨语言共享向量 —— 标准化 HTTP/原生准入、角色分派、canonical/legacy MIME 处理、有限 body 上限 |
| **NIP** | v0.12 + v0.13 | **0.12**：§7.5 **Phase-3 强制模式** —— 接收侧 `phase3_enforcement` 开关，把 Phase 1–2 的可选 CA 证明检查（assurance / `node_roles` / capabilities / OCSP staple）转为硬性 MUST；子集检查，仅在对应证书扩展存在时生效；`NIP-CERT-CAPABILITIES-EXCEEDED`。**0.13**：§7.6 **可移植 CA 与验证 profile** —— 确定性的验证/来源顺序、`if_configured` 与 fail-closed `required` 吊销模式、签名的确定性 CRL 语义、完整 CA-store 枚举、带认证的 `GET /v1/certificates` |
| **NDP** | v0.10 + v0.11 + v0.12 | **0.10（CR-0009）**：AnnounceFrame 新增 `cluster_epoch`（uint64，默认 1）；§9 最高 epoch 解析，同 epoch 脑裂 → `NDP-CLUSTER-SPLIT`；联邦传播 `(cluster_anchor, cluster_epoch, active_nid)`。**0.11（CR-0010）**：`bridge_inbound_protocols`；声明 `"bridge"` 的节点两数组 MUST 至少一非空。**0.12**：**Registry Conformance profile** —— 新增 `graph_seq` wire 字段 + 兼容行为；确定性签名体 canonicalization；有序的 signature/profile/replay/conflict/staleness 检查 |
| **NOP** | v0.8 + v0.9 | **0.8（CR-0009）**：`DelegateFrame.target_cluster_anchor` MUST 解析到集群当前活跃 Anchor（最高 `cluster_epoch`）；`anchor_failover` 时在途委托 MUST 在重试前重解析到 `successor_nid`；§8 租约续约语义规范化（`NOP-CLAIM-CONFLICT`）。**0.9**：**可移植 orchestrator profile** —— 确定性 DAG preflight 与合规调度；单次求值的 condition/input mapping；retry / timeout / cancellation / K-of-N / aggregate 规则 |
| **实现修复** | —— | **NOP 帧 wire key** 在 Go / Rust / TS / Java 对齐 NPS-5：`DelegateFrame` `task_id`→`parent_task_id`、`target_nid`\|`agent_nid`→`target_agent_nid`；`SyncFrame` `subtask_ids`→`wait_for`；`AlignStream` `sync_id`→`stream_id`、`source_nid`→`sender_nid`。纯合规修复；TS/Java 保留旧键解码回退 |
| **共享** | error-codes v1.8 · frame-registry v0.14 | CR-0009 的五个错误码（`NWP-ANCHOR-NOT-LEADER`、`NWP-ANCHOR-EPOCH-FENCED`、`NWP-BRIDGE-DIRECTION-UNSUPPORTED`、`NDP-CLUSTER-SPLIT`、`NIP-CERT-CAPABILITIES-EXCEEDED`）加上 profile 波的新增 |

**Daemons** —— ✅ `nps-registry` 已实现 CR-0009 最高 epoch 解析 + `NDP-CLUSTER-SPLIT`。仍待后续：`nps-ingress` 的完整 `TC-N2-*` / `TC-N2-HA-*` L2 向量覆盖；`nps-runner` 的 `SpawnSpec` OCI 镜像解析 + 租约续约边界情况。

**不在范围内（→ beta.1）**：NIP Phase-3 **flag day** 本身（把强制变为默认 MUST）；多区域 NPS Cloud CA（Phase 3）；超出合规向量的 QUIC 生产级加固。

---

## alpha.18 — 🚧 下一个（目标 2026-09）—— **Pre-beta.1 加固**

> **主题**：*beta.1 前加固* —— 对每个协议做生产加固并清理发布工程欠账，为 `v1.0.0-beta.1` 的 NIP Phase-3 flag day 铺路。不引入大范围功能波；唯一受控设计例外是 NPS-Dev#90，即 NWP stateful LLM context/delta contract，使模型输入 token 节省能被真实测量，而不只停留在传输层节省。
>
> **铁律（自 alpha.11）**：每个 alpha 五协议 **spec 与六 SDK 同步推进**，不允许 ".NET 参考先行" 缺口。下述每项都落 spec + go/java/python/rust/typescript/.NET + conformance 向量 + CN 译文 + 文档四面。

**目标版本**：NCP 0.12 / NWP 0.21 / NIP 0.14 / NDP 0.13 / NOP 0.10；`error-codes.md` 1.9 / `frame-registry.yaml` 0.15。

**当前基线（2026-08-12）**：
- ✅ NPS-1..NPS-5 中文规范已与英文在结构和语义上对齐（NPS-Dev#87）；alpha.18 不再背历史译债。
- ✅ alpha.17 wire-key 尾项已在 Dev 事实源收口，包括 Java/Rust `DelegateFrame` 对齐（NPS-Dev#86）。
- 🚧 alpha.18 首批：官方 NWP LLM usage/unary correlation（NPS-Dev#88）与 .NET nullable-`UInt64` NativeAOT resolver 修复（NPS-Dev#89）。
- 🚧 发布欠账：TypeScript 动态包版本、可维护的 Go 最低版本、删除式 standalone 物化、conformance fixture vendoring、registry 发布权限检查。

**逐协议加固**：

| 协议 | 版本 | 已有基线 | alpha.18 验收 |
|---|---|---|---|
| **NCP** | 0.12 | 已有 NopFrame、`ping_interval_ms`、dead-peer 规则和 native TCP/QUIC profile | 六 SDK 都执行 runtime keepalive timer 与确定性超时关连；定稿 QUIC connection migration、0-RTT 拒绝、流控/背压策略；共享 idle/dead-peer/oversized-Nop 向量 |
| **NWP** | 0.21 | 已有拓扑授权、Subscribe cursor、LLM DTO、可移植 native/HTTP server | 所有 server 强制执行授权与可续订 subscription；统一 stability/SLA/billing metadata；交付 #88；把 #90 写成经评审的 CR，覆盖 append/fork/reset/release、归属、过期、usage 与 streaming correlation，再落六 SDK |
| **NIP** | 0.14 | 已有 edge-mTLS profile、吊销模式、可移植 CA 验证和 Phase-3 开关 | 短寿证书续期互操作；OCSP/CRL timeout/stale/unknown 下 fail-closed；提供 advisory 迁移工具，报告 beta.1 Phase-3 会拒绝的输入 |
| **NDP** | 0.13 | 已有 resolve-time staleness、health、三跳联邦、registry profile 与 `cluster_epoch` | 持久化并恢复 sequence/epoch fence；用共享故障向量覆盖重启、分区、stale、同 epoch split、loop 与恢复；交付 #89 |
| **NOP** | 0.10 | 已有聚合策略、ACK/NAK 字段、result TTL 与可移植 orchestrator profile | 六 runtime 一致执行有界滑窗重放/逐出、TTL 过期、`weighted_first_k`、`merge_all`；新增丢包/乱序/重复/超时 conformance 向量 |

**执行关口**：
1. **P18-0 —— 事实清理**：关闭实际已发布的 issue；落 #88/#89；删除 TS 静态版本漂移；建立并测试 Go 1.23 下限；修正 EN/CN 发布手册。
2. **P18-1 —— Spec/design freeze**：每协议一份规范性 hardening delta；#90 通过 CR review；五协议及共享 registry bump；EN/CN 与共享正向/负向/故障向量同批落地。
3. **P18-2 —— Runtime parity**：六 SDK 全部执行 P18-1 行为。只移植字段/DTO 不算完成；timer、持久化、取消、过期、重放和故障路径都必须可运行。
4. **P18-3 —— 故障与打包关口**：六语言测试、NativeAOT、可用语言的 race/concurrency、故障向量、package dry-run、安全与依赖扫描，均在文档化最低工具链上通过。
5. **P18-4 —— 分发**：用删除语义和 distribution 排除物化 standalone；vendoring 各自执行的 conformance fixture；Dev→Release/SDK 达到零未解释漂移；再走常规 pre-release review。tag/publish 仍需单独批准。

**发布手册铁律**：crates 是含 `nps-conformance` 的 **8** 个；standalone 同步删除旧源码并保留明确的 distribution-only 文件；每个 standalone 都带自己执行的 conformance fixture；Maven 无 `zip` 时可用 Python `zipfile`；registry preflight 必须证明发布权限（`cargo owner --list`、npm granular 读写 token + publish 2FA bypass），不能只证明匿名读取。

**不在范围（→ beta.1）**：NIP Phase-3 flag day 本身；多区域 NPS Cloud CA（Phase 3）；1.0 spec 冻结。

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
- [x] Tier-3 BinaryVector v1 规范（CR-0008）
- [ ] Tier-3 MatrixTensor / 更多 dtype 扩展

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
