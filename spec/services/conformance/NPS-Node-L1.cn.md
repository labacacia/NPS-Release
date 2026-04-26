[English Version](./NPS-Node-L1.md) | 中文版

# NPS-Node-L1 合规测试套件

**Status**: Draft
**Version**: 0.1
**Date**: 2026-04-24
**Applies-To**: [NPS-Node Profile §3](../NPS-Node-Profile.cn.md) — Level 1 Basic
**Authors**: Ori Lynn / INNO LOTUS PTY LTD

> 本文档定义实现声明 NPS-Node Profile L1 合规所 MUST 通过的测试用例。用例与语言无关；
> 参考实现使用 .NET 10 + xUnit，位于 `impl/dotnet/tests/NPS.Daemon.Conformance.Tests/`。

---

## 1. 使用方式

1. 构建或安装 **被测实现**（**IUT**）。
2. 启动一个 **对端（peer）**—— 任意已通过 L1 的实现，或 .NET 参考 SDK。
3. 将 IUT 与 peer 配对，逐条跑完 §3 所有测试用例。
4. 用例通过当且仅当 **全部** 验收条件成立。
5. L1 认证 MUST 全 21 个用例通过；不接受部分声明。
6. 将 [`NPS-NODE-L1-CERTIFIED.md`](./NPS-NODE-L1-CERTIFIED.md) 复制到 IUT 仓库根目录，
   填完每个字段，用 IUT 的 root 私钥对声明块签名。

本次发布接受自声明。第三方认证（NPS Cloud CA）规划于 2027 Q1+ 的 L3 阶段，不在本套件范围。

---

## 2. 测试环境

| 项 | 值 |
|----|----|
| 网络 | 仅 loopback；用例不要求外部 DNS 或跨机路由 |
| Peer | 任一 L1 合规实现（推荐：`NPS.Core` + `NPS.NDP` + `NPS.NIP` + `NPS.NWP` NPS v1.0.0-alpha.3 或更高） |
| 时钟 | 与 peer 墙钟偏差 ≤ ±5 秒（ISO 8601 UTC） |
| 文件系统 | IUT 密钥库与 inbox 持久化可写目录 |
| Wire 编码 | Tier-1 JSON MUST 覆盖；Tier-2 MsgPack 用例推迟到 L2 |

每个用例从 IUT 干净状态开始：用例间 MUST 删除 IUT 密钥库 / 注册表 / inbox，除非用例显式标
注为 continuation。

---

## 3. 测试用例

每个用例列出 [NPS-Node Profile §3](../NPS-Node-Profile.cn.md) 的 Req ID、前置条件、操作
步骤、验收条件。

### 3.1 NCP — Wire 格式

#### TC-N1-NCP-01 — Tier-1 JSON 帧往返
**Req**：N1-NCP-01
**前置**：Peer 构造每个 L1 帧（HelloFrame、AnchorFrame、IdentFrame、AnnounceFrame、ResolveFrame、ActionFrame、CapsFrame、ErrorFrame）的一个合法实例。
**操作**：Peer 逐一发给 IUT；IUT 把解码后的帧再编码发回。
**通过条件**：
- IUT 解码每个帧无错。
- 再编码输出经 RFC 8785 JSON canonicalization 后与输入字节相同。

#### TC-N1-NCP-02 — Hello + Anchor 握手
**Req**：N1-NCP-02
**前置**：IUT loopback 监听；peer 空闲。
**操作**：Peer 打开 TCP 连接，发 HelloFrame，等 CapsFrame，然后发 AnchorFrame。
**通过条件**：
- IUT 用 CapsFrame 应答 HelloFrame，声明自己支持的 encoding tier。
- IUT ACK AnchorFrame 并缓存供后续帧解析。
- 回路在 loopback 上 ≤ 500 ms 完成。

#### TC-N1-NCP-03 — Loopback 监听默认
**Req**：N1-NCP-03
**前置**：IUT 启动无 `--listen` override。
**操作**：Peer 探测 `127.0.0.1:17433`。
**通过条件**：
- IUT 在默认地址上接受连接。
- IUT 不接受非 loopback 地址（如 `0.0.0.0:17433`）的连接，除非运营方显式打开配置。

#### TC-N1-NCP-04 — Tier-2 协商卫生
**Req**：N1-NCP-04
**前置**：IUT 未配置 Tier-2 支持。
**操作**：Peer 发 HelloFrame，只声明 Tier-2。
**通过条件**：
- IUT 用 CapsFrame 应答，列出 `tier1-json` 为支持的 tier。
- IUT 不静默回退到不支持的 tier；若无公共 tier，IUT 返 ErrorFrame with `NPS-CLIENT-NEGOTIATION-FAILED`（或 peer 可观察到的等价）。

### 3.2 NIP — 身份

#### TC-N1-NIP-01 — Root keypair 生成与权限
**Req**：N1-NIP-01
**前置**：空的密钥库目录。
**操作**：启动 IUT；定位持久化的 root 密钥文件。
**通过条件**：
- IUT 首次启动 MUST 生成且仅生成一对 Ed25519 keypair。
- POSIX 下私钥文件权限 `0600`；Windows 下 owner-only ACL。
- 后续启动复用同一把密钥（不重新生成）。

#### TC-N1-NIP-02 — IdentFrame 签名与验签
**Req**：N1-NIP-02
**前置**：Peer 持一个已知合法 IdentFrame；IUT 有自己的 root 密钥。
**操作**：(a) Peer 在握手后把自己的 IdentFrame 作为第一帧发出。(b) Peer 另开一连接，发一个签名被篡改一个字节的 IdentFrame。
**通过条件**：
- (a) IUT 接受合法 IdentFrame。
- (b) IUT 拒绝篡改的 IdentFrame，返 ErrorFrame 映射到 `NPS-AUTH-UNAUTHENTICATED`。
- (a) IUT 发给 peer 的出站 IdentFrame 在 IUT root 公钥下验签成功。

#### TC-N1-NIP-03 — NID 格式
**Req**：N1-NIP-03
**前置**：IUT 新初始化。
**操作**：读取 IUT 广播的 NID（通过 AnnounceFrame 或 CLI）。
**通过条件**：
- NID 匹配 `urn:nps:node:<authority>:<id>`，`<authority>` 非空，`<id>` 非空。
- NID 跨重启稳定（同一密钥库干净重启后不变）。

#### TC-N1-NIP-04 — Sub-NID 签发（L1 可选）
**Req**：N1-NIP-04
**前置**：IUT 支持 sub-NID 签发。
**操作**：不支持时跳过。否则托管 Agent 申请 session sub-NID。
**通过条件**：
- 若尝试：签发的 sub-NID 由 root 私钥签名，TTL 非零，在 peer 信任策略下验签通过。
- 若拒绝：IUT 返回良构的 `NPS-SERVER-NOT-IMPLEMENTED`，用例记为 **N/A** 而非失败。

### 3.3 NDP — 发现

#### TC-N1-NDP-01 — AnnounceFrame 携带 activation_mode
**Req**：N1-NDP-01
**前置**：IUT 托管至少一个 Agent。
**操作**：Peer 订阅 NDP；捕获 IUT 发出的下一个 AnnounceFrame。
**通过条件**：
- `activation_mode` 字段存在。
- 值精确为 `"ephemeral"`（L1 仅支持 `ephemeral`）。

#### TC-N1-NDP-02 — AnnounceFrame 签名
**Req**：N1-NDP-02
**前置**：Peer 已拿到 IUT IdentFrame 公钥。
**操作**：Peer 抓取 IUT 的一个 AnnounceFrame,验证签名。
**通过条件**：
- 签名在声明的 NID 公钥下验过。
- 对捕获帧的任一非签名字段翻一个字节的拷贝 MUST 验签失败。

#### TC-N1-NDP-03 — ResolveFrame 响应
**Req**：N1-NDP-03
**前置**：IUT 托管一个 Agent，NID 为 `A`。
**操作**：Peer 对 `A` 的 `nwp://` URL 发 ResolveFrame；再对未知 NID `Z` 发 ResolveFrame。
**通过条件**：
- `A` 的响应包含 Agent 物理端点与未过期 TTL。
- `Z` 的响应是 ErrorFrame with `NDP-RESOLVE-NOT-FOUND`。

#### TC-N1-NDP-04 — GraphFrame 订阅（L1 可选）
**Req**：N1-NDP-04
**前置**：IUT 的 GraphFrame 订阅能力。
**操作**：按能力跳过或尝试。完整行为在 L2 校验。
**通过条件**：
- 若尝试：IUT 返良构的初始 GraphFrame；若 L1 拒绝则 **N/A**。

### 3.4 NWP — Inbox 与交付

#### TC-N1-NWP-01 — Inbox 接受 ActionFrame
**Req**：N1-NWP-01
**前置**：IUT 托管 Agent `A`；inbox 空。
**操作**：Peer 向 `A` 发 ActionFrame。
**通过条件**：
- IUT 确认接受（L1 pull 模型下不要求立即 body 应答）。
- 后续 peer 对 `A` 的 NWP pull 返回该帧，字节相同。

#### TC-N1-NWP-02 — Inbox 跨重启持久
**Req**：N1-NWP-02
**前置**：在 TC-N1-NWP-01 完成的 pull 步骤之前。
**操作**：停止 IUT（POSIX `SIGTERM` / Windows 等价），等待优雅关闭。同一数据目录重启 IUT。
Peer 再 pull `A`。
**通过条件**：
- Pull 返回重启前未投递的 ActionFrame，字节相同。

#### TC-N1-NWP-03 — NWP pull 服务 inbox
**Req**：N1-NWP-03
**前置**：Agent `A` inbox 中有 3 个 pending 帧。
**操作**：Peer 顺序发 3 次 NWP pull。
**通过条件**：
- 第 1 次 pull 返回第 1（最旧）个帧。
- 第 2、3 次 pull 按 FIFO 返回第 2、3 个帧。
- 第 4 次 pull 在 1 s 内返回空结果（inbox 清空后不阻塞）。

#### TC-N1-NWP-04 — 100 QPS baseline
**Req**：N1-NWP-04
**前置**：Agent `A` inbox 预填 10 000 个小（≤ 1 KB）CapsFrame。
**操作**：Peer 持续 10 s 发 100 QPS pull。
**通过条件**：
- 零 pull 错误。
- 常规硬件（1 vCPU / 1 GB RAM baseline）99 分位请求时延 < 100 ms。
- Inbox drain 计数 == 发出请求数（无重复、无丢失）。

#### TC-N1-NWP-05 — Push 路径（L1 可选）
**Req**：N1-NWP-05
**前置**：IUT 的 push 能力。
**操作**：L1 未声明 push 时跳过。完整行为在 L2 校验。
**通过条件**：
- 若尝试：推送到 `activation_endpoint` 的帧字节相同地被接收；若 L1 拒绝则 **N/A**。

### 3.5 可观测性

#### TC-N1-OBS-01 — 每方向帧日志
**Req**：N1-OBS-01
**前置**：IUT 启动，日志输出到可捕获的目的地。
**操作**：Peer 发起一次请求/响应往返（入一个 AnnounceFrame，出一个 AnnounceFrame ACK）。
**通过条件**：
- 日志每个帧方向至少一条记录。
- 无帧方向漏记。

#### TC-N1-OBS-02 — 日志字段
**Req**：N1-OBS-02
**前置**：TC-N1-OBS-01 捕获的任意一条日志。
**操作**：解析日志（仅结构化格式 —— JSON、logfmt 或等价；纯散文日志视为失败）。
**通过条件**：
- 记录包含每个字段：ISO 8601 UTC 时间戳、方向（`in` / `out`）、帧类型、源 NID、目的 NID、字节数。
- 值类型正确（时间戳解析为 RFC 3339；字节数为非负整数）。

#### TC-N1-OBS-03 — 日志目的地灵活性
**Req**：N1-OBS-03
**前置**：IUT 配置暴露日志目的地选项。
**操作**：分别配置 IUT 输出到 (a) stdout 和 (b) 文件。
**通过条件**：
- 两个目的地均能捕获 TC-N1-OBS-01/02 的同样日志。
- L1 不要求投递到远程日志聚合器。

---

## 4. 结果 Manifest

一次合规运行产出一个 JSON manifest，汇总每个用例结果。manifest 嵌入到
[`NPS-NODE-L1-CERTIFIED.md`](./NPS-NODE-L1-CERTIFIED.md)：

```json
{
  "profile": "NPS-Node-L1",
  "profile_version": "0.1",
  "iut": {
    "name": "example-daemon",
    "version": "0.1.0",
    "nid": "urn:nps:node:example.com:host-01"
  },
  "peer": {
    "name": "nps-dotnet-reference",
    "version": "1.0.0-alpha.3"
  },
  "run": {
    "date": "2026-04-24T00:00:00Z",
    "environment": "linux-x64 / 1 vCPU / 1 GB"
  },
  "cases": [
    { "id": "TC-N1-NCP-01", "result": "pass" },
    { "id": "TC-N1-NCP-02", "result": "pass" }
    /* ... 19 more ... */
  ],
  "summary": { "pass": 21, "fail": 0, "skip": 0, "na": 0 }
}
```

**全部 21 个用例为 `pass` 或 `na` 时** 授予认证（N1-NIP-04、N1-NDP-04、N1-NWP-05 在 IUT 拒绝可选能力时可为 `na`）。

---

## 5. 参考套件位置

| 语言 | 路径 | 状态 |
|------|------|------|
| .NET 10（xUnit） | `impl/dotnet/tests/NPS.Daemon.Conformance.Tests/` | 计划中；随 NPS Daemon MVP 一起跟踪 |
| Python | `impl/python/tests/conformance/node_l1/` | TODO（Phase 2） |
| TypeScript | `impl/typescript/tests/conformance/node-l1/` | TODO（Phase 2） |

参考套件的测试命名 MUST 与上文 `TC-N1-*` ID 对齐，便于测试报告 1:1 映射到 §4 manifest。

---

## 6. 变更历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1 | 2026-04-24 | 初稿：21 个测试用例覆盖 NCP / NIP / NDP / NWP / 可观测性、paired-peer 方法论、结果 manifest schema |

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
