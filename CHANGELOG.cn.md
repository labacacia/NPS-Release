[English Version](./CHANGELOG.md) | 中文版

# 变更日志 —— NPS-Release

本仓库归档每个 NPS 套件版本的规范文档和 GitHub Pages 站点。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

---

## [Unreleased]

### 修复

- **`spec/NPS-2-NWP.md` — 幽灵字段 `auth.min_assurance_level` 修复**：NWP §4.1 散文承诺了通过 ActionSpec 上的 `auth.min_assurance_level` 做 per-action 保证级别覆盖，但 §4.6 ActionSpec 字段表未定义该字段。修复：新增 `min_assurance_level: string（可选）`平级字段；同步修正 §4.1 散文和 §15 v0.6 changelog 行。（NPS-RFC-0003）

- **`spec/error-codes.md` v0.9 → v1.0**：四个 NWP 拓扑错误码（`NWP-TOPOLOGY-UNAUTHORIZED`、`NWP-TOPOLOGY-UNSUPPORTED-SCOPE`、`NWP-TOPOLOGY-DEPTH-UNSUPPORTED`、`NWP-TOPOLOGY-FILTER-UNSUPPORTED`）已在 NPS-2 NWP §13 和 NPS-CR-0002 §3.5 中定义并引用，但 alpha.4 规范同步时漏写入 error-codes 注册表。本次补入注册，措辞与 NWP §13 定义保持一致。

---

## [1.0.0-alpha.4] —— 2026-04-30

### 规范

- **NPS-CR-0002 —— Implemented（pre-1.0 快速通道）**：在 Anchor Node 上保留查询类型
  `topology.snapshot` 和 `topology.stream`，在 NPS-AaaS-Profile L2 级别为强制要求。
  NWP 新增顶层 §12，涵盖两种查询类型；QueryFrame §6.1 和 SubscribeFrame §8.1 均新增
  可选 `type` 字段；DiffFrame §8.2 `event_type` 枚举扩展了拓扑事件类型。
  `spec/error-codes.md` 新增四个错误码。新增 §14.7 拓扑读回安全节。
  AaaS-Profile §4.3 新增 L2-08，要求维护成员注册表的 Anchor Node 实现两种查询类型。
  新合规测试套件 `spec/services/conformance/NPS-Node-L2.md` v0.1，含七个
  `TC-N2-AnchorTopo-*` / `TC-N2-AnchorStream-*` 测试用例；
  配套 `NPS-NODE-L2-CERTIFIED.md` 自我声明模板。

- **NPS-RFC-0002 —— 原型已落地（状态：Draft）**：`IdentFrame` 新增可选
  `cert_format`（`"v1-proprietary"` 默认 | `"v2-x509"`）和 `cert_chain`（base64url DER）字段
  —— 非破坏性双信任扩展；v1 验证方忽略新字段，行为不变。`spec/error-codes.md` 新增四个
  NIP 错误码（`NIP-CERT-FORMAT-INVALID`、`NIP-CERT-EKU-MISSING`、
  `NIP-CERT-SUBJECT-NID-MISMATCH`、`NIP-ACME-CHALLENGE-FAILED`）。
  RFC 仍为 Draft，晋升 Proposed/Accepted 等待跨 SDK 移植波次完成 + IANA PEN
  （`nid-assurance-level` 当前使用临时 OID `1.3.6.1.4.1.99999`）。

### 版本变更（相对 alpha.3）

| 文档 | alpha.3 | alpha.4 |
|------|---------|---------|
| NPS-1 NCP | v0.6 | v0.6（不变）|
| NPS-2 NWP | v0.7 | v0.8 |
| NPS-3 NIP | v0.5 | v0.5（不变）|
| NPS-4 NDP | v0.5 | v0.5（不变）|
| NPS-5 NOP | v0.4 | v0.4（不变）|
| NPS-AaaS-Profile | v0.3 | v0.4 |
| NPS-Node-Profile | v0.1 | v0.1（不变）|
| NPS-Node-L2 合规 | n/a | v0.1（新增）|
| frame-registry | v0.9 | v0.9（不变）|
| error-codes | v0.8 | v0.9 |
| status-codes | v0.4 | v0.4（不变）|
| token-budget | v0.2 | v0.2（不变）|

### GitHub Pages

- README 及 Pages 站点内容全面更新至 `v1.0.0-alpha.4`。

---

## [1.0.0-alpha.3] —— 2026-04-26

### 规范 —— 内容大版本

这是首个**内容厚重**的 alpha 周期版本：规范文档从 alpha.2 的 "Proposed
+ 占位伴生件" 进入 "Proposed + 四个已 Accepted 的 Change-style 文档 +
显著扩展的接口面"。

- **目录拍平**：删除 `spec/protocols/`；`NPS-{0..5}-*.md` 直接住在
  `spec/` 下，与开发主仓 `LabAcacia/nps` 一致。形如
  `spec/protocols/NPS-1-NCP.md` 的外部链接应改为 `spec/NPS-1-NCP.md`。

- **NPS-CR-0001 —— Implemented**：把原 `Gateway Node`（NWP）拆为
  **Anchor Node**（集群控制平面 + NOP 路由 —— 角色继承重命名）与
  **Bridge Node**（NPS↔非-NPS 协议翻译 —— 全新）。Wire 值
  `node_type: "gateway"` 移除，解析器 MUST 拒绝。NWP §2.1 重写并新增
  Anchor + Bridge 完整子节；AaaS-Profile §2 把 Gateway 重命名为 Anchor，
  新增 §2A Bridge Node。NDP `Announce` 新增 `node_kind` /
  `cluster_anchor` / `bridge_protocols`。承担相反方向的历史适配器
  `NPS-{mcp,a2a,grpc}-bridge`（现 `NPS-*-ingress`）记录在
  `spec/cr/NPS-CR-0001-anchor-bridge-split.md` 中。

- **NPS-RFC-0001 —— Accepted（Phase 1）**：NCP 原生模式 8 字节连接
  前导 `b"NPS/1.0\n"`。`NPS-1-NCP.md` 新增 §2.6.1。新增
  `NCP-PREAMBLE-INVALID` 错误码 + 新 `PROTO` 状态分类的
  `NPS-PROTO-PREAMBLE-INVALID`。帧类型字节 `0x4E` 已保留。

- **NPS-RFC-0003 —— Accepted（Phase 1）**：Agent 身份三级保证等级
  （`anonymous` / `attested` / `verified`），用于反爬与信任分流。
  `NPS-3-NIP.md` 新增 §5.1.1；`IdentFrame` 新增可选
  `assurance_level`；NWM 新增可选 `min_assurance_level`（顶层 +
  per-action 覆盖）。新增错误码 `NIP-ASSURANCE-MISMATCH` /
  `NIP-ASSURANCE-UNKNOWN` / `NWP-AUTH-ASSURANCE-TOO-LOW`。X.509
  critical extension 翻转需等 RFC-0002（alpha.4）。

- **NPS-RFC-0004 —— Accepted（Phase 1）**：append-only NID 声誉
  日志（Agent 版 Certificate Transparency）。`NPS-3-NIP.md` 新增
  §5.1.2，定义 12 字段签名条目结构、8 项 `incident` 枚举、5 级
  `severity` 阶梯，以及 RFC 8785（JCS）双签名规则。新增错误码
  `NIP-REPUTATION-ENTRY-INVALID` / `NIP-REPUTATION-LOG-UNREACHABLE`，
  新增状态码 `NPS-DOWNSTREAM-UNAVAILABLE`。Merkle 树 + STH +
  inclusion proof 推迟到 alpha.4。

- **NPS-Node Profile**（`spec/services/NPS-Node-Profile.md`、
  `spec/services/conformance/NPS-Node-L1.md` + L1-CERTIFIED 模板）：
  节点侧 L1/L2/L3 合规规范，与服务侧 AaaS Profile 正交。NDP
  `Announce` 新增 `activation_mode`（`ephemeral` / `resident` /
  `hybrid`）+ `activation_endpoint` + `spawn_spec_ref`。L1 附带 21
  个 `TC-N1-*` 测试用例 + 一个 Ed25519 自证模板。

- **`spec/cr/`** 归档目录引入：1.0 之前的轻量级 Change Request
  归档，与正式 RFC 轨道互补。当前收录 `NPS-CR-0001`（已实现）与
  `NPS-CR-0002-anchor-topology-queries`（依赖 CR-0001；排到 alpha.4）。

### 版本升级（vs alpha.2）

| 文档 | alpha.2 | alpha.3 |
|------|---------|---------|
| NPS-1 NCP | v0.5 | v0.6 |
| NPS-2 NWP | v0.5 | v0.7 |
| NPS-3 NIP | v0.3 | v0.5 |
| NPS-4 NDP | v0.3 | v0.5 |
| NPS-5 NOP | v0.4 | v0.4（无变化）|
| NPS-AaaS-Profile | v0.2 | v0.3 |
| NPS-Node-Profile | n/a | v0.1（新）|
| frame-registry | v0.5 | v0.9 |
| error-codes | v0.5 | v0.8 |
| status-codes | v0.2 | v0.4 |
| token-budget | v0.2 | v0.2（无变化）|

### GitHub Pages

- README + Pages 站点源同步指向拍平后的 `spec/` 布局、新的
  `spec/cr/` 与 `spec/rfcs/` 索引、以及全文 v1.0.0-alpha.3。

---

## [1.0.0-alpha.2] —— 2026-04-19

### Changed

- 规范文档与主仓 `LabAcacia/nps` 重新同步：
  - 11 份规范文档全部从 `Draft` → `Proposed`。
  - 版本升级：NPS-0 → v0.3、NPS-1 → v0.5、NPS-2 → v0.5、NPS-3 → v0.3、NPS-4 → v0.3、NPS-5 → v0.4、NPS-Roadmap → v0.3、error-codes → v0.5、status-codes → v0.2、token-budget → v0.2、NPS-AaaS-Profile → v0.2。
  - `frame-registry.yaml` 升级至 v0.5，所有 `protocol_version` 字段同步更新。
- `Depends-On` 交叉引用已同步。

### Deferred

- alpha.2 版本的 GitHub Pages 内容刷新推迟到发布后一并更新。

---

## [1.0.0-alpha.1] —— 2026-04-10

首次归档规范文档和 GitHub Pages 站点。

[1.0.0-alpha.4]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.4
[1.0.0-alpha.3]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.3
[1.0.0-alpha.2]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.2
[1.0.0-alpha.1]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.1
