[English Version](./README.md) | 中文版

# Neural Protocol Suite (NPS) — 协议规范

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Release](https://img.shields.io/badge/release-v1.0.0--alpha.14-orange.svg)](CHANGELOG.cn.md#100-alpha14--2026-06-26)
[![NCP](https://img.shields.io/badge/NCP-v0.8-5b8cff.svg)]()
[![NWP](https://img.shields.io/badge/NWP-v0.14-4af0b0.svg)]()
[![NIP](https://img.shields.io/badge/NIP-v0.10-7b61ff.svg)]()
[![NDP](https://img.shields.io/badge/NDP-v0.9-f0a050.svg)]()
[![NOP](https://img.shields.io/badge/NOP-v0.7-ff8c42.svg)]()

> **版本：** 1.0.0-alpha.14 | **最新已发布版本：** 1.0.0-alpha.14 | **状态：** Released | **许可证：** Apache 2.0
>
> Copyright 2026 INNO LOTUS PTY LTD — LabAcacia Open Source

**NPS (Neural Protocol Suite)** 是面向 AI Agent 与神经模型的协议族，用语义优先、Agent-native 的 wire protocol 替代传统 HTTP/REST 堆栈中的高损耗部分。

---

## NPS 仓库

| 仓库 | 角色 | 语言 / 形态 |
|------|------|-------------|
| **[NPS-Release](https://github.com/labacacia/NPS-Release)**（本仓库） | 协议规范；所有 SDK 的权威 SSoT | Markdown / YAML |
| [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | 参考实现：frame codec、NWP、NIP CA、conformance | C# / .NET 10 |
| [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py) | Python SDK | Python 3.11+ |
| [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts) | Node/browser SDK | TypeScript / Node 22+ |
| [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java) | JVM SDK | Java 21+ |
| [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust) | Rust SDK | Rust stable |
| [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go) | Go SDK | Go 1.25+ |
| [nip-ca-server](https://github.com/labacacia/nip-ca-server) | 可自托管的 NIP 证书颁发机构 | C# / .NET 10 + PostgreSQL |
| [nps-daemons](https://github.com/labacacia/nps-daemons) | 标准 NPS 拓扑的参考 daemon bundle：`npsd`、`nps-runner`、`nps-ingress`、`nps-registry` | C# / .NET 10 |

命名说明：进程级 Internet ingress daemon 名为 `nps-ingress`。它不同于规范层已退役的 **Gateway Node** 逻辑角色；CR-0001 已将该逻辑角色替换为 **Anchor Node** 和 **Bridge Node**。

`.NET` SDK 提供底层 CA library（`LabAcacia.NPS.NIP`）；独立的 [`nip-ca-server`](https://github.com/labacacia/nip-ca-server) 仓库将其包装为可部署服务。`nps-cloud-ca` 与 `nps-ledger` 是 `innolotus` 组织下的私有 Layer-3 trust-anchor daemon，会随 NPS Cloud GA 公开。

---

## 为什么需要 NPS？

HTTP、REST、GraphQL 为人类浏览器设计。AI Agent 使用它们时会遇到四类成本：

| 问题 | 影响 |
|------|------|
| 每个响应重复 schema | Token 浪费、延迟升高 |
| 无原生 Agent 身份 | 鉴权外挂、信任链不完整 |
| 语义留给 prompt 解释 | Prompt 复杂、幻觉风险更高 |
| 单请求模型 | 缺少原生 streaming 与任务编排 |

NPS 在 wire 层解决这些问题：一次性 schema anchor、每跳 Ed25519 身份、帧内语义标注，以及统一 DAG task frame。

---

## 协议族

| 协议 | 类比 | 版本 | 摘要 |
|------|------|------|------|
| **NCP** — Neural Communication Protocol | Wire / Framing | v0.8 | 二进制 frame、JSON/MsgPack 双编码、streaming、native-mode preamble、有界 Hello、`NopFrame` keepalive、TLS/mTLS native transport binding |
| **NWP** — Neural Web Protocol | HTTP / NCP | v0.14 | 语义请求/响应、AnchorFrame schema cache、Memory / Action / Anchor / Bridge node、SubscribeFrame、manifest versioning、`bridge_target` schema、NCP native serving |
| **NIP** — Neural Identity Protocol | TLS / PKI | v0.10 | Ed25519 身份、证书生命周期、CA、OCSP、签名 CRL、live revocation hook、X.509/ACME、assurance level、reputation、edge-mTLS profile |
| **NDP** — Neural Discovery Protocol | DNS | v0.9 | 节点公告、签名记录、GraphFrame 拓扑快照、liveness 字段、resolve-time staleness、联邦转发 |
| **NOP** — Neural Orchestration Protocol | SMTP / MQ | v0.7 | DAG 编排、delegation、streaming result、AlignStream ack/NAK、Saga 补偿、webhook HMAC、result TTL、L3 runtime claim 语义 |

**依赖链：** `NCP ← NWP ← NIP ← NDP` / `NCP + NWP + NIP ← NOP`

### alpha.14 release 重点

alpha.14 release 文档跟踪 Banyan 集成反馈与 SDK 对齐工作：

- 各 SDK 的类型化远程 NIP CA client，覆盖 discovery、CRL、Ed25519 注册以及 RFC-0002 X.509 注册流程；
- native-mode NWP serving helper，让节点可以直接在已建立的 NCP session 上处理 `QueryFrame` / `ActionFrame`；
- TC-N1/TC-N2 conformance catalog、manifest 与 CI/self-certification validator；
- .NET 侧 live revocation、native NCP TLS/mTLS hook、有界 native handshake、签名 CRL 与 transport-neutral observability 加固。

---

## 文档

### 协议规范

| 文档 | 说明 |
|------|------|
| [NPS-0 Overview](./spec/NPS-0-Overview.cn.md) | 协议族总览 |
| [NPS-1 NCP](./spec/NPS-1-NCP.cn.md) | Wire format、frame header、encoding tiers |
| [NPS-2 NWP](./spec/NPS-2-NWP.cn.md) | Neural Web Protocol |
| [NPS-3 NIP](./spec/NPS-3-NIP.cn.md) | Neural Identity Protocol |
| [NPS-4 NDP](./spec/NPS-4-NDP.cn.md) | Neural Discovery Protocol |
| [NPS-5 NOP](./spec/NPS-5-NOP.cn.md) | Neural Orchestration Protocol |

### 参考文档

| 文档 | 说明 |
|------|------|
| [Frame Registry](./spec/frame-registry.yaml) | 机器可读 frame type 注册表 |
| [Error Codes](./spec/error-codes.cn.md) | 统一协议错误码命名空间 |
| [Status Codes](./spec/status-codes.cn.md) | NPS 原生状态码 + HTTP 映射 |
| [Token Budget](./spec/token-budget.cn.md) | CGN token budget 规范 |
| [Version Matrix](./spec/version-matrix.yaml) | 机器可读 suite/spec 版本 oracle |
| [Node Conformance](./spec/conformance/README.md) | TC-N1/TC-N2/TC-N3 合规入口 |
| [Roadmap](./docs/roadmap.cn.md) | Phase 0–4 路线图 |

### 服务规范

| 文档 | 说明 |
|------|------|
| [AaaS Profile](./spec/services/NPS-AaaS-Profile.cn.md) | Agent-as-a-Service 合规 profile |
| [Node Profile](./spec/services/NPS-Node-Profile.cn.md) | 节点侧合规 profile |

---

## Frame Type Namespace 速查

| 范围 | 协议 | Frames |
|------|------|--------|
| `0x01-0x0F` | **NCP** | Anchor(0x01), Diff(0x02), Stream(0x03), Caps(0x04), Align(0x05, deprecated), Hello(0x06), Nop(0x07) |
| `0x10-0x1F` | **NWP** | Query(0x10), Action(0x11), Subscribe(0x12) |
| `0x20-0x2F` | **NIP** | Ident(0x20), Trust(0x21), Revoke(0x22) |
| `0x30-0x3F` | **NDP** | Announce(0x30), Resolve(0x31), Graph(0x32) |
| `0x40-0x4F` | **NOP** | Task(0x40), Delegate(0x41), Sync(0x42), AlignStream(0x43) |
| `0xFE` | System | ErrorFrame |

完整机器可读 registry 见 [`spec/frame-registry.yaml`](./spec/frame-registry.yaml)。

---

## 贡献

贡献流程见 [CONTRIBUTING.cn.md](./CONTRIBUTING.cn.md)。破坏性 `spec/` 变更必须先开 RFC issue。

---

## 许可证

Copyright 2026 INNO LOTUS PTY LTD

Licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE) for details.

- **规范与参考实现** — LabAcacia open source
- **商业服务（NPS Cloud）** — INNO LOTUS PTY LTD 运营
- **Primary author** — Ori Lynn
