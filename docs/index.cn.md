# NPS — 面向 AI Agent 的 Wire Protocol

> **Neural Protocol Suite** — 为 AI Agent 和神经模型量身设计的完整互联网协议栈。
>
> 1.0.0-alpha.5 · Apache 2.0 · [English](index.md)

---

## 问题

今天的 AI Agent 通过 HTTP、REST、HTML 访问 Web — 这些协议是为**人类浏览器**设计的，而不是为按 token 推理的模型设计的。

| 问题 | 代价 |
|------|------|
| 每次响应重复 Schema | Token 浪费，延迟升高 |
| 没有原生的 Agent 身份 | 外挂的认证，缺失信任链 |
| 语义解析留给 Agent | Prompt 臃肿，幻觉增多 |
| 单次请求模型 | 没有原生的流式 / 任务编排 |

## 答案

NPS 在**协议层面**同时解决这四个问题：

- **一次性 Schema Anchor** — 服务端发布一次 `AnchorFrame`，Agent 用内容寻址 id 引用
- **每一跳都有 Ed25519 身份** — `NipIdentity` 是一等公民，不是附加品
- **帧内语义注解** — 语义在帧里，不在散文里
- **统一的 DAG 任务帧** — 多 Agent 工作流，不必重造 MQ 或 Temporal

---

## 体验一下

```bash
# Python
pip install nps-lib==1.0.0a5

# TypeScript
npm install @labacacia/nps-sdk@1.0.0-alpha.5

# Rust
cargo add nps-sdk@=1.0.0-alpha.5

# Go
go get github.com/labacacia/NPS-sdk-go@v1.0.0-alpha.5

# Java (Gradle)
implementation("com.labacacia.nps:nps-java:1.0.0-alpha.5")

# .NET
dotnet add package LabAcacia.NPS.Core --version 1.0.0-alpha.5
```

---

## 探索

- [**总览**](overview.cn.md) — NPS 是什么、为何存在、面向谁
- [**协议族**](protocols.cn.md) — 五层协议（NCP / NWP / NIP / NDP / NOP）一览
- [**SDK**](sdks.cn.md) — 六种语言的安装与快速开始
- [**路线图**](roadmap.cn.md) — Phase 0 → Phase 4

---

## 状态

**v1.0.0-alpha.5** — NWP 错误码全集 + RFC-0004 Phase 3（STH gossip 联邦）。全 30 个 NWP wire 错误码现已作为常量在六种 SDK 中发布。`nps-ledger` 新增 STH gossip 联邦（`GET /v1/log/gossip/sth`），含签名验证与单调性检查。新增 `NPS-SERVER-UNSUPPORTED` 状态码（HTTP 501）。Python、TypeScript、Java 三种 SDK 修复 `AssuranceLevel.from_wire("")` / `fromWire("")` 的 spec §5.1.1 行为。规范：NCP v0.6、NWP v0.10、NIP v0.6、NDP v0.6、NOP v0.4。.NET、Python、TypeScript、Java、Rust、Go 六种参考实现全部覆盖 NCP + NWP + NIP + NDP + NOP。**NIP CA Server** 在 [`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server)。**NPS Daemons** bundle（`npsd` + `nps-runner` + `nps-gateway` + `nps-registry` + `nps-ledger`）在 [`labacacia/nps-daemons`](https://github.com/labacacia/nps-daemons)。Layer-3 信任锚 daemon `nps-cloud-ca` 在 `innolotus` 组织私有，跟 NPS Cloud GA 一起公开（2027 Q1+）。

---

## 链接

- [GitHub — NPS-Release](https://github.com/labacacia/NPS-Release) · 权威协议规范
- [CONTRIBUTING](https://github.com/labacacia/NPS-Release/blob/main/CONTRIBUTING.md) · 如何提案
- [LICENSE](https://github.com/labacacia/NPS-Release/blob/main/LICENSE) · Apache 2.0

版权所有 © 2026 INNO LOTUS PTY LTD — LabAcacia 开源实验室。

---

📖 教程、参考资料和运维指南，请访问 [NPS Wiki](https://github.com/labacacia/NPS-Release/wiki)。
