[English Version](./CHANGELOG.md) | 中文版

# 变更日志 —— NPS-Release

本仓库归档每个 NPS 套件版本的规范文档和 GitHub Pages 站点。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

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

[1.0.0-alpha.3]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.3
[1.0.0-alpha.2]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.2
[1.0.0-alpha.1]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.1
