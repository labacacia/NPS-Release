[English](./README.md) | 中文版

# Neural Protocol Suite (NPS) — 协议规范

> **版本：** 1.0.0-alpha.4 | **状态：** Proposed | **许可证：** Apache 2.0
>
> Copyright 2026 INNO LOTUS PTY LTD — LabAcacia 开源实验室

**NPS（Neural Protocol Suite，神经协议套件）** 是专为 AI Agent 和神经模型设计的完整互联网基础协议族，旨在以语义优先、Agent 原生的方式替代 HTTP/REST 技术栈。

---

## NPS 仓库导航

| 仓库 | 职责 | 语言 |
|------|------|------|
| **[NPS-Release](https://github.com/labacacia/NPS-Release)**（本仓库） | 协议规范 — 所有 SDK 的权威源 | Markdown / YAML |
| [NPS-sdk-dotnet](https://github.com/labacacia/NPS-sdk-dotnet) | 参考实现 — 帧编解码、Memory Node 中间件、NIP CA | C# / .NET 10 |
| [NPS-sdk-py](https://github.com/labacacia/NPS-sdk-py) | 异步 Python SDK | Python 3.11+ |
| [NPS-sdk-ts](https://github.com/labacacia/NPS-sdk-ts) | ESM + CJS 双输出 Node/浏览器 SDK | TypeScript |
| [NPS-sdk-java](https://github.com/labacacia/NPS-sdk-java) | JVM SDK | Java 21+ |
| [NPS-sdk-rust](https://github.com/labacacia/NPS-sdk-rust) | 异步 Rust SDK | Rust stable |
| [NPS-sdk-go](https://github.com/labacacia/NPS-sdk-go) | Go SDK | Go 1.23+ |
| [nip-ca-server](https://github.com/labacacia/nip-ca-server) | NIP 证书颁发机构 —— 单 Docker 自托管 CA，签发 NID 证书 | C# / .NET 10 + PostgreSQL |
| [nps-daemons](https://github.com/labacacia/nps-daemons) | 参考部署二进制 —— `npsd`、`nps-runner`、`nps-gateway`、`nps-registry`（NPS 标准三层中的 Layer 1 + Layer 2） | C# / .NET 10 |

.NET SDK 提供底层 CA 库（`LabAcacia.NPS.NIP`）；独立仓
[`nip-ca-server`](https://github.com/labacacia/nip-ca-server) 把它
封装成可部署服务。仓库 `example/` 目录下另收录 5 个冻结的参考移植
（Python / TypeScript / Java / Rust / Go）。

Layer-3 信任锚 daemon（`nps-cloud-ca` 和 `nps-ledger`）在 GitHub
`innolotus` 组织下作为**私有**仓存放，跟 **NPS Cloud** GA 一起公开
（计划 2027 Q1+）。今天就要自托管 CA，用
[`labacacia/nip-ca-server`](https://github.com/labacacia/nip-ca-server)。

---

## 为什么需要 NPS？

现有 Web 协议（HTTP、REST、GraphQL）是为人类浏览器设计的，AI Agent 在使用时面临根本性问题：

| 问题 | 影响 |
|------|------|
| Schema 随每次响应重复传输 | Token 浪费，延迟增加 |
| 无原生 Agent 身份概念 | 认证机制外挂，无信任链 |
| 语义解释留给 Agent 处理 | Prompt 复杂度高，幻觉风险 |
| 单次请求-响应模型 | 无原生流式传输或任务编排 |

NPS 在协议层解决以上四个问题：一次性 Schema 锚点、Ed25519 身份内嵌每跳、帧内语义标注、统一 DAG 任务帧。

---

## 协议族

| 协议 | 类比 | 版本 | 说明 |
|------|------|------|------|
| **NCP** — Neural Communication Protocol | Wire / 帧格式 | v0.4 | 二进制帧格式、双层编码（JSON/MsgPack）、流式传输 |
| **NWP** — Neural Web Protocol | HTTP | v0.7 | 语义请求/响应、AnchorFrame Schema 缓存、Memory/Action/Anchor/Bridge 节点 |
| **NIP** — Neural Identity Protocol | TLS / PKI | v0.5 | Ed25519 身份、证书生命周期、CA、OCSP、CRL |
| **NDP** — Neural Discovery Protocol | DNS | v0.5 | 节点公告、签名记录、图遍历 |
| **NOP** — Neural Orchestration Protocol | SMTP / MQ | v0.4 | DAG 任务编排、委托、流式结果 |

**依赖关系：** `NCP ← NWP ← NIP ← NDP` / `NCP + NWP + NIP ← NOP`

---

## 文档

### 协议规范文档

| 文档 | 说明 |
|------|------|
| [NPS-0 总览](./spec/NPS-0-Overview.cn.md) | 套件总览，阅读入口 |
| [NPS-1 NCP](./spec/NPS-1-NCP.cn.md) | 帧格式、帧头、编码分层 |
| [NPS-2 NWP](./spec/NPS-2-NWP.cn.md) | Neural Web Protocol |
| [NPS-3 NIP](./spec/NPS-3-NIP.cn.md) | Neural Identity Protocol |
| [NPS-4 NDP](./spec/NPS-4-NDP.cn.md) | Neural Discovery Protocol |
| [NPS-5 NOP](./spec/NPS-5-NOP.cn.md) | Neural Orchestration Protocol |

### 参考文档

| 文档 | 说明 |
|------|------|
| [帧注册表](./spec/frame-registry.yaml) | 机器可读帧类型注册表 |
| [错误码](./spec/error-codes.cn.md) | 统一错误码命名空间 |
| [状态码](./spec/status-codes.cn.md) | NPS 原生状态码 + HTTP 映射 |
| [Token Budget](./spec/token-budget.cn.md) | NPT Token Budget 规范 |
| [路线图](./spec/NPS-Roadmap.cn.md) | Phase 0–4 开发路线图 |

### 服务层规范

| 文档 | 说明 |
|------|------|
| [AaaS Profile](./spec/services/NPS-AaaS-Profile.cn.md) | Agent-as-a-Service 合规规范（Anchor Node、Bridge Node、Vector Proxy Layer、L1/L2/L3 合规） |

---

## 核心架构决策

| 决策 | 结论 | 原因 |
|------|------|------|
| 默认端口 | **17433** | 全协议族共用，帧类型码天然路由 |
| 传输模式 | HTTP 模式 + 原生模式 | HTTP 模式防火墙友好；原生模式高性能 |
| Schema 所有权 | Node 发布 AnchorFrame | Node 是数据模型拥有者，Agent 引用 ID |
| Token 计量 | **NPT**（NPS Token） | 跨模型统一计量单位 |
| 主签名算法 | **Ed25519** | 性能优先，适合 Agent 高频验签 |
| 默认编码 | **MsgPack**（Tier-2） | 生产环境约 60% 体积压缩 |
| 默认帧大小 | 64 KB（EXT=0） | 扩展：4 GB（EXT=1） |
| 最大 DAG 节点数 | 32 | 防止资源耗尽 |
| 最大委托链深度 | 3 层 | 防止无限递归委托 |
| 最大图谱遍历深度 | 5 层 | X-NWP-Depth 上限 |
| AnchorFrame TTL | 3600 秒 | 平衡缓存命中率与 Schema 更新及时性 |

---

## 帧类型命名空间（速查）

| 范围 | 协议 | 帧 |
|------|------|------|
| `0x01–0x0F` | **NCP** | Anchor(0x01)、Diff(0x02)、Stream(0x03)、Caps(0x04)、Align(0x05，已废弃)、Hello(0x06) |
| `0x10–0x1F` | **NWP** | Query(0x10)、Action(0x11) |
| `0x20–0x2F` | **NIP** | Ident(0x20)、Trust(0x21)、Revoke(0x22) |
| `0x30–0x3F` | **NDP** | Announce(0x30)、Resolve(0x31)、Graph(0x32) |
| `0x40–0x4F` | **NOP** | Task(0x40)、Delegate(0x41)、Sync(0x42)、AlignStream(0x43) |
| `0xFE`      | 系统   | ErrorFrame — 跨协议统一错误帧 |

完整机器可读清单参见 [`spec/frame-registry.yaml`](./spec/frame-registry.yaml)。

---

## 贡献

参见 [CONTRIBUTING.md](./CONTRIBUTING.md) 了解贡献流程和破坏性变更的 RFC 要求。

Issue 前缀：
- `spec:` — 规范问题与设计讨论
- `impl:` — 参考实现 Bug
- `sdk:` — SDK 相关（Python / TypeScript / ……）
- `docs:` — 文档改进

`spec/` 目录下的破坏性变更 **必须** 先开 RFC Issue 讨论。

---

## 许可证

Copyright 2026 INNO LOTUS PTY LTD

使用 Apache License, Version 2.0 授权。详见 [LICENSE](./LICENSE)。

- **规范与参考实现** — LabAcacia 开源
- **商业服务（NPS Cloud）** — 由 INNO LOTUS PTY LTD 运营
- **主要作者** — Ori Lynn
