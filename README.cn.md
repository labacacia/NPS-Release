[English](./README.md) | 中文版

# Neural Protocol Suite (NPS) — 协议规范

> **版本：** 1.0.0-alpha.1 | **状态：** Draft | **许可证：** Apache 2.0
>
> Copyright 2026 INNO LOTUS PTY LTD — LabAcacia 开源实验室

NPS（Neural Protocol Suite，神经协议套件）是专为 AI Agent 和神经模型设计的完整互联网基础协议族，旨在以语义优先、Agent 原生的方式替代 HTTP/REST 技术栈。

---

## 为什么需要 NPS？

现有 Web 协议（HTTP、REST、GraphQL）是为人类浏览器设计的，AI Agent 在使用时面临根本性问题：

| 问题 | 影响 |
|------|------|
| Schema 随每次响应重复传输 | Token 浪费，延迟增加 |
| 无原生 Agent 身份概念 | 认证机制外挂，无信任链 |
| 语义解释留给 Agent 处理 | Prompt 复杂度高，幻觉风险 |
| 单次请求-响应模型 | 无原生流式传输或任务编排 |

NPS 在协议层解决以上四个问题。

---

## 协议族

| 协议 | 类比 | 版本 | 说明 |
|------|------|------|------|
| **NCP** — Neural Communication Protocol | Wire / 帧格式 | v0.4 | 二进制帧格式、编码分层、流式传输 |
| **NWP** — Neural Web Protocol | HTTP | v0.4 | 语义请求/响应、AnchorFrame Schema 缓存 |
| **NIP** — Neural Identity Protocol | TLS / PKI | v0.2 | Ed25519 身份、证书生命周期、CA |
| **NDP** — Neural Discovery Protocol | DNS | v0.2 | 节点公告、图遍历、能力查找 |
| **NOP** — Neural Orchestration Protocol | SMTP / MQ | v0.3 | DAG 任务编排、委托、流式结果 |

**依赖关系：** `NCP ← NWP ← NIP ← NDP` / `NCP + NWP + NIP ← NOP`

---

## 文档

### 协议规范文档

| 文档 | 说明 |
|------|------|
| [NPS-0 总览](./doc/protocols/NPS-0-Overview.md) | 套件总览，阅读入口 |
| [NPS-1 NCP](./doc/protocols/NPS-1-NCP.md) | 帧格式、帧头、编码分层 |
| [NPS-2 NWP（中文）](./doc/protocols/NPS-2-NWP.md) | Neural Web Protocol |
| [NPS-2 NWP（English）](./doc/protocols/NPS-2-NWP.en.md) | Neural Web Protocol |
| [NPS-3 NIP](./doc/protocols/NPS-3-NIP.md) | Neural Identity Protocol |
| [NPS-4 NDP](./doc/protocols/NPS-4-NDP.md) | Neural Discovery Protocol |
| [NPS-5 NOP（中文）](./doc/protocols/NPS-5-NOP.md) | Neural Orchestration Protocol |
| [NPS-5 NOP（English）](./doc/protocols/NPS-5-NOP.en.md) | Neural Orchestration Protocol |

### 参考文档

| 文档 | 说明 |
|------|------|
| [帧注册表](./doc/frame-registry.yaml) | 机器可读帧类型注册表 |
| [错误码](./doc/error-codes.md) | 统一错误码命名空间 |
| [状态码](./doc/status-codes.md) | NPS 原生状态码 + HTTP 映射 |
| [Token Budget](./doc/token-budget.md) | NPT Token Budget 规范 |
| [路线图](./doc/NPS-Roadmap.md) | Phase 0–4 开发路线图 |

### 服务层规范

| 文档 | 说明 |
|------|------|
| [AaaS Profile（中文）](./doc/services/NPS-AaaS-Profile.md) | Agent-as-a-Service 合规规范 |
| [AaaS Profile（English）](./doc/services/NPS-AaaS-Profile.en.md) | Agent-as-a-Service 合规规范 |

---

## 核心架构决策

| 决策 | 结论 | 原因 |
|------|------|------|
| 默认端口 | 17433（全协议族共用） | 帧类型码天然路由，简化部署 |
| 传输模式 | HTTP 模式 + 原生模式 | HTTP 模式防火墙友好；原生模式高性能 |
| Schema 所有权 | Node 发布 AnchorFrame | Node 是数据模型拥有者，Agent 只读引用 |
| Token 计量 | NPT（NPS Token） | 跨模型统一计量单位 |
| 主签名算法 | Ed25519 | 性能优先，适合 Agent 高频验签 |
| 默认编码 | MsgPack（Tier-2） | 生产环境约 60% 体积压缩 |
| 最大 DAG 节点数 | 32 | 防止资源耗尽攻击 |
| 最大委托链深度 | 3 层 | 防止无限递归委托 |

---

## SDK 实现

| 语言 | 包名 | 仓库 |
|------|------|------|
| .NET / C# | `NPS.Core`（NuGet） | https://github.com/labacacia/NPS-sdk-dotnet |
| Python | `nps-sdk`（PyPI） | https://github.com/labacacia/NPS-sdk-py |
| Go | `github.com/labacacia/nps-sdk-go` | https://github.com/labacacia/NPS-sdk-go |
| TypeScript | `@labacacia/nps-sdk`（npm） | https://github.com/labacacia/NPS-sdk-ts |
| Java | `com.labacacia:nps-java`（Maven Central） | https://github.com/labacacia/NPS-sdk-java |
| Rust | `nps-sdk`（crates.io） | https://github.com/labacacia/NPS-sdk-rust |

---

## 贡献

参见 [CONTRIBUTING.md](./CONTRIBUTING.md) 了解贡献流程和破坏性变更的 RFC 要求。

## 许可证

Copyright 2026 INNO LOTUS PTY LTD

使用 Apache License, Version 2.0 授权。详见 [LICENSE](./LICENSE)。
