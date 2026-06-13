[English Version](./NPS-RFC-0005-reputation-policy-enforcement.md) | 中文版

---
**RFC 编号**: NPS-RFC-0005
**标题**: 声誉策略强制执行
**状态**: Active
**作者**: Ori Lynn <iamzerolin@gmail.com>（LabAcacia）
**Shepherd**: Ori Lynn（pre-1.0 快速通道，见 `spec/cr/README.md`）
**创建日期**: 2026-05-19
**最后更新**: 2026-05-28
**通过日期**: 2026-05-19（pre-1.0 快速通道）
**激活日期**: 2026-05-28（v1.0.0-alpha.9 — DefaultReputationPolicyEvaluator、AnchorNodeMiddleware 及 X-NWP-Ident 保证级别提取均已发布）
**取代**: _无_
**被取代**: _无_
**影响规范**: NPS-2 NWP、NPS-3 NIP、spec/services/NPS-AaaS-Profile.md、spec/error-codes.md
**影响 SDK**: .NET、Python、TypeScript、Java、Rust、Go
---

# NPS-RFC-0005：声誉策略强制执行

## 1. 摘要

定义 `reputation_policy`（声誉策略）的线格式及强制执行生命周期。该策略是 Anchor Node
的每节点配置，控制其如何查询 NPS-RFC-0004 声誉日志，对入站 Agent 请求作出
**放行 / 限流 / 拒绝 / 封禁** 四种准入决策。

本 RFC 将 RFC-0004 §4.4 中的草图转化为规范的、可实现的规格，添加了 Agent 侧在
`IdentFrame.metadata` 中声明日志背书的对应字段，并引入 SDK 中间件必须实现的
`ReputationPolicyEvaluator` 接口。

## 2. 动机

RFC-0004 定义了日志线格式，但有意将强制执行留给各节点自行决定。AaaS Profile L2-09
要求（SHOULD 在准入时查询日志）缺乏标准化的策略 Schema，导致节点实现之间以及工具
（NPS Probe、NPS Studio）之间无法互操作。

本 RFC 填补的具体空白：

- **无共享策略 Schema**：两家 AaaS 运营商配置"拒绝有重大限流违规的 Agent"时，当前
  使用互不兼容的临时格式，日志运营商无法发布合规声明。
- **声誉决策无标准错误码**：RFC-0004 现有的 `NWP-AUTH-REPUTATION-BLOCKED`
  是单一无差别错误；限流、拒绝、封禁需要不同的响应，调用方才能实现正确的重试逻辑。
- **无 Agent 侧日志背书**：Agent 没有标准方式声明其接受哪些日志运营商的查询，
  联邦日志发现语义模糊。

## 3. 非目标

- **不**定义什么构成可报告事件（见 RFC-0004 §4.2）。
- **不**要求全局唯一的权威日志或日志联邦方案。
- **不**规定节点如何缓存日志查询结果（仅规定规范性最低要求）。
- **不**将 `IdentFrame` 签名扩展至覆盖 `metadata.reputation_policy`——
  metadata 依据 NIP §5.1 信任边界规则保持无签名状态。

## 4. 详细设计

### 4.1 节点侧策略：NWM / `AnchorNodeOptions` 中的 `reputation_policy`

Anchor Node 可在 NWM 清单的顶级 `reputation_policy` 键下发布策略块，
同一结构也驱动 `AnchorNodeOptions.ReputationPolicy` SDK 配置对象。

#### 4.1.1 线格式（JSON / YAML）

```json
{
  "reputation_policy": {
    "enabled": true,
    "log_sources": [
      "https://log.nps.example.com/v1/reputation",
      "https://consortium-log.nps.org/v1/reputation"
    ],
    "min_assurance_level": "attested",
    "cache_ttl_seconds": 300,
    "ban_ttl_seconds": 3600,
    "throttle_on": [
      { "incident": "rate-limit-violation", "severity": ">=minor", "within_days": 7 }
    ],
    "reject_on": [
      { "incident": "tos-violation",    "severity": ">=major",  "within_days": 30 },
      { "incident": "scraping-pattern", "severity": ">=major",  "within_days": 30 }
    ],
    "ban_on": [
      { "incident": "cert-revoked", "severity": ">=minor" },
      { "incident": "fraud",        "severity": ">=major" }
    ]
  }
}
```

#### 4.1.2 字段定义

| 字段 | 类型 | 必填 | 默认值 | 描述 |
|------|------|------|--------|------|
| `enabled` | bool | 否 | `true` | 为 `false` 时策略块被解析但跳过执行，适用于干运行审计 |
| `log_sources` | URL 数组 | 启用时必填 | — | 有序的 RFC-0004 合规日志运营商基础 URL 列表；按顺序查询，首个成功响应即采用 |
| `min_assurance_level` | string | 否 | `"anonymous"` | RFC-0003 最低保证级别，`"anonymous"` / `"attested"` / `"verified"` 之一；低于此级别的请求在日志查询前即被拒绝，返回 `NWP-ASSURANCE-MISMATCH` |
| `cache_ttl_seconds` | uint32 | 否 | `300` | 每 NID 日志查询结果缓存时长，0 表示不缓存 |
| `ban_ttl_seconds` | uint32 | 否 | `3600` | `ban_on` 规则触发后 NID 的封禁时长；封禁状态保存在进程内存中，节点重启后清除 |
| `throttle_on` | 规则数组 | 否 | `[]` | 触发限流（HTTP 429）的规则；任一匹配即触发 |
| `reject_on` | 规则数组 | 否 | `[]` | 触发硬拒绝（HTTP 403）的规则；在 `ban_on` 之后评估 |
| `ban_on` | 规则数组 | 否 | `[]` | 触发时限封禁（HTTP 403 + 封禁缓存条目）的规则；最先评估 |

#### 4.1.3 规则格式

`throttle_on`、`reject_on`、`ban_on` 中的每条规则均为对象：

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `incident` | string | 是 | RFC-0004 §4.2 事件类型，或 `"*"` 匹配任意事件 |
| `severity` | string | 是 | 严重性谓词，格式为 `">=<级别>"` 或精确值 `"<级别>"`；严重性从低到高：`info` / `minor` / `moderate` / `major` / `critical` |
| `within_days` | uint32 | 否 | 若设置，仅考虑 `timestamp` 在此天数内的日志条目；缺省表示不限时间 |
| `count` | uint32 | 否 | 触发规则所需的最少匹配条目数，默认为 1 |

**严重性谓词**：`">=moderate"` 匹配 `moderate`、`major` 和 `critical`；精确形式 `"major"` 仅匹配 `major`。

#### 4.1.4 执行顺序

评估器 MUST 按以下顺序应用规则：

1. **`min_assurance_level` 检查** — 若 Agent 保证级别（来自已验证的 IdentFrame）低于配置的最低要求，立即拒绝并返回 `NWP-ASSURANCE-MISMATCH`，跳过日志查询。
2. **封禁缓存检查** — 若请求方 NID 在本地封禁缓存中且 TTL 未到期，无需查询日志，直接返回 `NWP-REPUTATION-BANNED`。
3. **日志查询** — 向 `log_sources` 查询请求方 NID 的条目；若在 `cache_ttl_seconds` 内则使用缓存结果。日志不可达时，实现 MAY 回退到上一次缓存结果，或应用可配置的 `on_log_unavailable` 策略（`allow` | `deny`，默认 `allow`）。
4. **`ban_on` 评估** — 若任一规则触发，将 NID 写入封禁缓存并持续 `ban_ttl_seconds`，然后返回 `NWP-REPUTATION-BANNED`。
5. **`reject_on` 评估** — 若任一规则触发，返回 `NWP-REPUTATION-REJECTED`。
6. **`throttle_on` 评估** — 若任一规则触发，返回 `NWP-REPUTATION-THROTTLED`，附带 `Retry-After: 60` 响应头。
7. **放行** — 在响应头中设置 `X-NWP-Reputation-Status: clean`，继续正常请求处理。

评估器在步骤 4–6 之间 MUST NOT 因部分匹配而短路；每组规则独立按顺序评估。同时匹配 `ban_on` 和 `reject_on` 的 NID 以封禁结果为准（步骤 4 优先）。

### 4.2 Agent 侧声明：`IdentFrame.metadata` 中的 `reputation_policy`

Agent 可在 `IdentFrame.metadata` 中包含 `reputation_policy` 对象，声明其背书的日志运营商及对声誉查询的同意偏好。该字段**无签名**（依据 NIP §5.1 信任边界规则），因此仅供参考。

```json
{
  "reputation_policy": {
    "log_sources": [
      "https://log.nps.example.com/v1/reputation"
    ],
    "consent": "public"
  }
}
```

| 字段 | 类型 | 描述 |
|------|------|------|
| `log_sources` | URL 数组 | Agent 背书的日志运营商；节点 MUST NOT 将其作为自身查询的权威许可列表，仅作为日志发现提示 |
| `consent` | string | `"public"`（Agent 同意任何节点查询其声誉）或 `"private"`（Agent 倾向于将查询限制在直接对等方）；仅供参考，节点 MAY 不受此字段约束进行查询 |

**信任边界**：节点 MUST NOT 使用 `IdentFrame.metadata.reputation_policy` 来提升信任级别、绕过 `min_assurance_level` 检查，或替代节点运营商配置的 `log_sources`。

### 4.3 SDK 接口：`ReputationPolicyEvaluator`

每个 SDK MUST 提供实现以下契约的评估器类型（以 .NET 表达；其他语言使用惯用等价物）：

```csharp
public interface IReputationPolicyEvaluator
{
    Task<ReputationDecision> EvaluateAsync(
        string requesterNid,
        AssuranceLevel assuranceLevel,
        ReputationPolicy policy,
        CancellationToken ct = default);
}

public enum ReputationOutcome { Accept, Throttle, Reject, Ban }

public sealed record ReputationDecision(
    ReputationOutcome Outcome,
    ReputationPolicyRule? MatchedRule,
    string? ErrorCode);
```

`AnchorNodeMiddleware` 在请求流水线的身份验证之后、动作分发之前调用
`IReputationPolicyEvaluator.EvaluateAsync`。评估器支持注入（DI / 构造函数参数），
允许自定义实现（例如缓存后端、测试桩）。

### 4.4 错误码

| 错误码 | HTTP 状态 | NPS 状态 | 描述 |
|--------|-----------|---------|------|
| `NWP-REPUTATION-THROTTLED` | 429 | `NPS-CLIENT-RATE-LIMITED` | 声誉策略限流（`throttle_on` 规则匹配）；响应 MUST 包含 `Retry-After: 60` 头 |
| `NWP-REPUTATION-REJECTED` | 403 | `NPS-AUTH-FORBIDDEN` | 声誉策略拒绝（`reject_on` 规则匹配）；响应体包含 `matched_incident` 和 `matched_severity` |
| `NWP-REPUTATION-BANNED` | 403 | `NPS-AUTH-FORBIDDEN` | 声誉策略封禁（`ban_on` 规则匹配或封禁缓存条目生效）；响应 SHOULD 包含 `X-NWP-Ban-Expires` Unix 时间戳 |

现有的 RFC-0004 错误码 `NWP-AUTH-REPUTATION-BLOCKED` 已被上述三个码取代；实现 MUST
发出新码，旧码可作为弃用别名保留一个 alpha 周期。

### 4.5 NWM 发布

当 `reputation_policy.enabled` 为 `true` 时，节点 MUST 将完整策略对象（包括 `log_sources`）
发布到 `/.nwm` 清单的 `reputation_policy` 键下，使调用方在发送请求前即可了解节点的
执行立场。节点 MUST NOT 在 NWM 中发布封禁缓存状态或逐 NID 查询结果。

### 4.6 AaaS Profile 对齐

AaaS Profile L2-09 更新为：

> **L2-09（RFC-0005 更新）**：SHOULD 在 NWM 中配置 `reputation_policy`（`enabled: true`）
> 并至少指定一个 `log_sources` 条目。L2 的**最低推荐策略**为：
> ```json
> {
>   "ban_on":    [{ "incident": "cert-revoked", "severity": ">=minor" }],
>   "reject_on": [
>     { "incident": "tos-violation",  "severity": ">=major", "within_days": 30 },
>     { "incident": "fraud",          "severity": ">=major" }
>   ],
>   "throttle_on": [
>     { "incident": "rate-limit-violation", "severity": ">=minor", "within_days": 7 }
>   ]
> }
> ```

## 5. 安全性考量

### 5.1 日志不可达

当所有 `log_sources` 均不可达时，评估器应用 `on_log_unavailable` 策略。默认值 `allow`
的目的是避免攻击者通过关闭日志运营商来阻断所有请求。高安全部署 SHOULD 设置
`on_log_unavailable: deny` 并接受对应的可用性权衡。

### 5.2 缓存污染

日志查询结果按 NID 缓存。能够污染 DNS 的攻击者可将 `log_sources` URL 重定向至
提供"干净"假结果的服务器。实现 MUST 为 `log_sources` URL 验证 TLS 证书链，
在日志 URL 受运营商控制时 SHOULD 固定日志运营商公钥。

### 5.3 声誉漂白

被封禁的 NID 可通过获取新 NID（新密钥对 + 新 CA 注册）来规避声誉执行，即经典的女巫攻击问题。
缓解措施：
- `min_assurance_level: "verified"` 提高获取新 NID 的成本。
- CA 运营商可在账户层面独立限制注册频率。
- 联盟日志运营商可跨 NID 关联行为模式。

### 5.4 隐私

日志查询会向日志运营商披露请求方 NID，这在公联邦部署中是有意为之。
在 NDP `org-private` Profile 下的节点 SHOULD 运营本地日志镜像或聚合查询，
以减少单次请求的披露程度。

## 6. 实现分阶段计划

| Alpha | 范围 |
|-------|------|
| **alpha.8** | RFC-0005 规范（本文档）；六个 SDK 中的 `ReputationPolicyEvaluator` + `AnchorNodeOptions.ReputationPolicy`；NWM `reputation_policy` 发布；三个新错误码 |
| **alpha.9** | `IdentFrame.metadata.reputation_policy` Agent 侧声明；更新 AaaS Profile L2-09 文本；NPS Probe 的 NWM 策略存在性合规检查 |

## 7. 开放问题

1. **`on_log_unavailable` 默认值**：本 RFC 为可用性考虑默认为 `allow`。AaaS Profile L2 是否应对某些事件类型（如 cert-revoked）要求 SHOULD `deny`，并使用本地缓存的吊销列表？本地吊销缓存无需硬性 `deny` 即可解决此问题，但增加了实现复杂性。

2. **封禁持久化**：本 RFC 规定封禁状态保存在进程内存中，节点重启后清除。`org-private` 和 `public-federated` 注册表 Profile（NDP §7.3）是否应要求封禁状态跨重启持久化？SQLite 持久化封禁存储将与 `graph_seq` 持久化要求保持一致。

3. **多日志共识**：当 `log_sources` 有多个条目且它们不一致时（日志 A 显示干净，日志 B 显示重大违规），本 RFC 采用最严格结果。规范是否应允许 `quorum` 模式，要求 `ceil(N/2)` 个日志达成一致后规则才触发？

## 8. 变更日志

| 版本 | 日期 | 变更 |
|------|------|------|
| 草案 | 2026-05-19 | 初始草案 |

---

*版权所有：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
