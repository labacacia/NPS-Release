# 路线图

> [English](roadmap.md) | 中文版

NPS 的路径分为四个 Phase — 从草案规范走向正式标准。当前发布 — **v1.0.0-alpha.1** — 为六种 SDK 收尾 Phase 1，打开 Phase 2。

---

## 节奏

每个 Phase 分三段：

```
① Spec Sprint（2 周）   — 本 Phase 所有规范冻结
② Impl Sprint（6–8 周） — 实现、测试、文档
③ Review Gate（1 周）   — 社区 / 内部评审，决定是否进入下一 Phase
```

### 版本约定

| 版本号 | 含义 |
|--------|------|
| `v0.x-draft`  | 内部草稿，可破坏性变更 |
| `v0.x-alpha`  | 公开预览，API 不稳定 |
| `v0.x-beta`   | 功能完整，欢迎外部测试 |
| `v1.0`        | 规范冻结，生产可用 |

---

## Phase 0 — 规范统一（2026 Q2）— ✅ 完成

目标：建立 NPS 完整规范骨架，统一帧空间和命名，输出可供社区评论的 v0.2-draft。

- ✅ `NPS-0-Overview.md` v0.2
- ✅ `NPS-1-NCP.md` v0.4（传输双模、可配帧大小、ErrorFrame）
- ✅ `NPS-2-NWP.md` v0.4（Node 发布 AnchorFrame、NPT）
- ✅ `NPS-3-NIP.md` v0.2（metadata 字段、NPS 状态码）
- ✅ `NPS-4-NDP.md` v0.2
- ✅ `NPS-5-NOP.md` v0.3
- ✅ `frame-registry.yaml` v0.2（含 ErrorFrame 0xFE）
- ✅ `error-codes.md` v0.2 + `status-codes.md` v0.1 + `token-budget.md` v0.1
- ✅ `services/NPS-AaaS-Profile.cn.md` v0.1（Gateway Node、Vector Proxy Layer、L1/L2/L3 合规要求）

---

## Phase 1 — 核心实现（2026 Q3）— ✅ 已交付

目标：NCP + NWP + NIP + NDP + NOP 在六种参考语言中生产可用。六语言版本的 NIP CA Server。

### SDK

| 语言 | 包名 | 状态 |
|------|------|------|
| .NET       | `NPS.SDK` / `NPS.Core` / `NPS.NOP` | ✅ v1.0.0-alpha.1 已发布 |
| Python     | `nps-sdk`                          | ✅ v1.0.0-alpha.1 已发布（162 tests, 97% coverage）|
| TypeScript | `@labacacia/nps-sdk`               | ✅ v1.0.0-alpha.1 已发布（139 tests, 98.75% coverage）|
| Java       | `com.labacacia:nps-sdk`            | ✅ v1.0.0-alpha.1 已发布（87 tests）|
| Rust       | `nps-sdk`                          | ✅ v1.0.0-alpha.1 已发布（88 tests）|
| Go         | `github.com/labacacia/nps/impl/go` | ✅ v1.0.0-alpha.1 已发布（75 tests）|

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

- ✅ `NPS.Core` — 帧编解码、AnchorCache、EXT 帧头
- ✅ `NPS.NWP` — Memory Node 中间件（SQL Server / PostgreSQL），284 integration tests
- ✅ `NPS.NOP` — DAG 校验器 + 编排引擎，委托链深度限制（NPS-5 §8.2）、callback_url SSRF 防护（§8.4）、指数退避重试，429 tests
- ✅ `NPS.NIP` — CA 库：密钥生成、证书签发 / 吊销、OCSP、CRL

完成标准：
- ✅ SDK 单元测试覆盖率 ≥ 90%
- ✅ Memory Node `QueryFrame` 往返集成测试通过
- ✅ NIP CA Server 一键 Docker Compose 启动文档可用

---

## Phase 2 — 生态扩展（2026 Q4）— 🚧 进行中

目标：对现有生态（MCP、A2A）的适配器，更丰富的 SDK 示例，Tier-2 MsgPack 生产硬化。

- ✅ TypeScript SDK 已交付（提前到 Phase 1）
- ✅ Go SDK 已交付（提前到 Phase 1）
- ⬜ `compat/mcp-bridge/` — NWP Memory Node ↔ MCP Resource 适配器
- ⬜ `compat/a2a-bridge/` — NOP `TaskFrame` ↔ A2A Task 适配器
- ⬜ Tier-2 MsgPack 基准报告（体积 ≤ JSON 的 50%）
- ⬜ 首个参考产品 — **NPS Studio**（人类可视化调试器）+ **NPS Probe**（Agent Coder 合规检查 CLI）

完成标准：
- ⬜ `NDP.ResolveFrame` 通过 DNS TXT 解析 `nwp://` 到物理端点
- ⬜ NOP 编排器可端到端执行 3 节点 DAG
- ⬜ Claude Desktop 通过 `mcp-bridge` 访问 NWP Memory Node

---

## Phase 3 — 生态验证（2027 Q1–Q2）

目标：真实场景 PoC，NPS Cloud CA v1.0，事实标准基础。

- ⬜ **NPS Cloud CA v1.0** — 多区域 HA，实时 OCSP，Professional Plan
- ⬜ LangChain / AutoGen / CrewAI 集成适配包
- ⬜ FinTech PoC（Open Banking 场景，跨组织 `TrustFrame`）
- ⬜ 车联网 PoC（设备 NID，`StreamFrame` 实时遥测）
- ⬜ 公开 Token 节约基准报告
- ⬜ NIP CA Server OSS v1.0（PostgreSQL + Web Admin UI）
- ⬜ GitHub Stars ≥ 500

---

## Phase 4 — 标准化（2027 Q3 起）

目标：推动 NPS 成为 W3C / IETF 正式标准；冻结 NPS 1.0。

- ⬜ 多厂商联合支持声明（≥ 3 家）
- ⬜ W3C WebAI Community Group 提案
- ⬜ IETF Internet-Draft（NCP + NWP 核心规范）
- ⬜ NPS 1.0 规范冻结
- ⬜ ISO/IEC JTC 1 评估
- ⬜ Tier-3 MatrixTensor 规范

---

## 风险登记册（节选）

| ID  | 风险 | 概率 | 影响 | 缓解 |
|-----|------|------|------|------|
| R01 | 规范变更导致实现返工 | 高 | 高 | 规范先冻结再实现；破坏性变更走 RFC |
| R03 | Token 节约效果不及预期（<30%）| 中 | 高 | Phase 1 起测基准；AnchorFrame 命中率是关键 |
| R04 | NIP CA 私钥安全事件 | 低 | 极高 | 预留 HSM 接口；强制年度轮换 |
| R07 | W3C / IETF 标准化周期过长 | 高 | 低 | 通过 OSS 推进事实标准；正式 RFC 次之 |

完整登记册：[NPS-Roadmap.md](https://github.com/labacacia/NPS-Release/blob/main/spec/NPS-Roadmap.cn.md)。

---

## 下一步

- [总览](overview.cn.md) — NPS 是什么
- [协议族](protocols.cn.md) — 五层协议
- [SDK](sdks.cn.md) — 选一门语言
