[English Version](./token-budget.md) | 中文版

# Cognon Budget 规范

**Version**: 0.5
**Date**: 2026-05-09

---

## 1. 概述

Cognon Budget 机制允许 Agent 在请求中声明本次操作的最大 token 消耗上限。Node 据此裁剪响应字段、限制返回条数或拒绝超预算请求。

为解决不同 LLM 的 token 计算差异，NPS 引入 **Cognon（CGN）** 作为标准化计量单位。

---

## 2. Cognon（CGN）

### 2.1 定义

Cognon（CGN）是 NPS 协议族内部的标准 token 计量单位。各 LLM 的原生 token 通过汇率转换为 CGN。

CGN 定义为**两个命名 profile**，其合规性要求互不重叠（issue #40）。在线传递的每一个 CGN 值 MUST 明确关联到其中**一个**且只有一个 profile；对端 MUST NOT 混用两者。

| Profile | 用途 | 使用场景 |
|---------|------|----------|
| **CGN-Estimate** | 估算、预算提示、遥测、可采样流程 | `X-NWP-Budget` 强制、CapsFrame 的 `token_est`、推送流逐事件 `cgn_est` 上报 |
| **CGN-Billing** | 商业结算、争议与冲销处理 | 计费的计量与对端之间交换的签名计量记录 |

#### CGN-Estimate（估算级）

- Tokenizer 来源 MAY 为 `declared_tokenizer`（NPS-3-NIP §5.1）或更高等级。
- 允许使用 §2.2 的字节数 fallback。
- 允许采样（详见 §6.2）。
- 与模型原生计数之间允许 ±5 % 的汇率漂移。
- 记录无需签名，无需审计日志集成。

#### CGN-Billing（结算级）

发出 CGN-Billing 记录的 Node MUST **同时**满足以下全部要求：

- 所用 tokenizer MUST 达到 [NPS-3-NIP §5.1](./NPS-3-NIP.cn.md#trust-boundary-for-unsigned-metadata-normative--closes-issue-39) 定义的 `verified_tokenizer` 等级。`declared_tokenizer`、`observed_tokenizer_profile` 以及 §2.2 的字节数 fallback **均禁止**作为计费输入。
- 每一条计量记录 MUST 由发起 Node 的 NID 签名，并 MUST 持久化到与 NOP §8.3（以及在已部署场景下 NPS-RFC-0004 logging）兼容的审计日志中。
- MUST NOT 使用采样。每一个被计费的 CGN 值 MUST 逐条精确计算，不允许估算。
- 计数 MUST 确定且非近似；§2.3 的 ±5 % 漂移容差**不**适用，因为计费费率属于精确合同条款，而非估算值。
- 争议与冲销 MUST 仅凭审计日志即可独立支撑，不得依赖任何易失的 Node 进程态。
- 汇率表版本 MUST 由双方在会话开始时刻（或更早）互相 pin 定，并记录在签名计量记录中。

发出以 CGN 计价的商业收费时，Node MUST 在线传输与响应头中（详见 §4.2）将其标记为 CGN-Billing。以 CGN-Estimate 形式或不带 profile 标记呈现的收费均**不合规**结算用途，对端 MAY 直接拒付争议，且无须援引 tokenizer 信任等级。

### 2.2 默认计算方法（Fallback）—— 仅适用于 CGN-Estimate

当 tokenizer 无法确定时，**CGN-Estimate** MAY 回退至以下公式：

```
CGN = ceil(UTF-8_bytes / 4)
```

此公式基于主流 LLM tokenizer 的平均行为（英文约 4 bytes/token，中文约 3 bytes/token），作为最保守的估算基线。

字节数 fallback 在任何情况下 MUST NOT 用于 **CGN-Billing**。若 Node 无法为某条将被计费的请求解析出 `verified_tokenizer`，MUST 拒绝为该请求发出 CGN-Billing 记录，并采取以下两种处理之一：(a) 将该路面降级为 CGN-Estimate（仅作不可计费遥测）；(b) 以计费类错误拒绝该请求。

### 2.3 汇率表（CGN Exchange Rates）

Node 实现 SHOULD 内置常见模型的 CGN 汇率：

| 模型族 | Tokenizer | 1 原生 Token ≈ CGN | 说明 |
|--------|-----------|---------------------|------|
| OpenAI GPT-4 / GPT-4o | `cl100k_base` | 1.0 | 基准参照 |
| Anthropic Claude | Claude tokenizer | 1.05 | 略高于 GPT-4 |
| Google Gemini | SentencePiece | 0.95 | 略低于 GPT-4 |
| Meta LLaMA 3 | `llama3-tokenizer` | 1.02 | 接近基准 |
| Mistral | SentencePiece | 0.98 | 接近基准 |
| Default（未知模型）| UTF-8 / 4 | 1.0 | Fallback（仅适用于 CGN-Estimate）|

汇率表随 NPS 版本更新维护。实现 MAY 通过配置覆盖内置汇率。

**Profile 适用范围。** 上表对 **CGN-Estimate** 是 normative 的，并允许表内值与模型原生计数之间存在文档化的 ±5 % 漂移。对 **CGN-Billing**，双方对端 MUST 互相 pin 定一个特定的表版本（在会话开始时刻或更早，并记录到签名计量记录中），且 MUST 使用与之匹配的、由 `verified_tokenizer` 得出的原生计数；±5 % 容差**不**适用。`Default（未知模型）` 行仅适用于 CGN-Estimate；CGN-Billing 没有 fallback 行。

---

## 3. Tokenizer 解析链

Agent 发起请求时，Node 按以下优先级确定 tokenizer：

```
1. Agent 显式声明（X-NWP-Tokenizer 头）
   ↓ 未声明
2. 从 Agent 配置/IdentFrame 自动匹配
   ↓ 匹配失败
3. 使用默认计算方法（UTF-8 bytes / 4）
```

### 3.1 显式声明（优先级最高）

Agent 在请求头中声明 tokenizer：

```
X-NWP-Tokenizer: cl100k_base
```

Node MUST 识别声明的 tokenizer 并使用对应算法计算 token。若 Node 不支持该 tokenizer，SHOULD 回退到自动匹配。

### 3.2 自动匹配

Node 根据 IdentFrame 中的元数据推断 Agent 使用的模型族：

- `IdentFrame.metadata.model_family`：如 `"openai/gpt-4o"`、`"anthropic/claude-4"`
- `IdentFrame.metadata.tokenizer`：如 `"cl100k_base"`

若 IdentFrame 包含以上字段，Node 使用对应的 tokenizer。

> **估算专用警示（normative —— issue #39）。** `metadata.model_family` 与 `metadata.tokenizer` 抵达节点时均属于 [NPS-3-NIP §5.1 —— 未签名 `metadata` 的信任边界](./NPS-3-NIP.cn.md#trust-boundary-for-unsigned-metadata-normative--closes-issue-39) 定义的 tokenizer 三层信任模型中的 **`declared_tokenizer`** 层；§3.1 的 `X-NWP-Tokenizer` 请求头与之同等信任级别。本节自动匹配出的取值 MUST 仅作为**估算提示**使用，MUST NOT 驱动计费、结算、配额提升、声誉评分或任何安全相关决策。结算级与策略级流程 MUST 改用 `verified_tokenizer`（CA 或平台背书）信号；`observed_tokenizer_profile` 仅可用于 Node 内部滥用检测的回退。基于 `declared_tokenizer` 计费或授予配额提升的 Node 不合规。

### 3.3 默认 Fallback

无法确定 tokenizer 时，使用 `ceil(UTF-8_bytes / 4)` 计算 CGN。

---

## 4. 请求与响应

### 4.1 请求头

| 头 | 必填 | 描述 |
|----|------|------|
| `X-NWP-Budget` | 可选 | 最大 CGN 预算（uint32）|
| `X-NWP-Tokenizer` | 可选 | Agent 使用的 tokenizer 标识 |

### 4.2 响应头

| 头 | Profile | 描述 |
|----|---------|------|
| `X-NWP-Tokens` | CGN-Estimate | 本响应实际消耗的 CGN（估算级）|
| `X-NWP-Tokens-Native` | CGN-Estimate | 本响应原生 token 消耗（若已知 tokenizer）|
| `X-NWP-Tokenizer-Used` | 两者 | Node 实际使用的 tokenizer 标识 |
| `X-NWP-Tokens-Profile` | 两者 | 取值为 `estimate` 或 `billing`。缺省或 `estimate` MUST 由对端视为 CGN-Estimate。 |
| `X-NWP-Billing-Record` | CGN-Billing | 指向本响应签名计量记录的引用（URI 或内容哈希）。当且仅当本响应按 CGN-Billing 计费时 MUST 存在。 |
| `X-NWP-Billing-Tokenizer-Tier` | CGN-Billing | MUST 为 `verified_tokenizer`。缺省 → 不可计费。 |

同时缺失 `X-NWP-Billing-Record` 与 `X-NWP-Billing-Tokenizer-Tier` 的响应 MUST 由对端解释为 CGN-Estimate，无论双方是否存在任何商业约定；Node MUST NOT 基于仅含 CGN-Estimate 信号的响应进行结算。

### 4.3 超预算处理

当响应将超过 `X-NWP-Budget` 时：

1. Node SHOULD 优先裁剪响应（减少返回字段或条数），使结果在预算内
2. 若无法裁剪（如单条记录已超预算），Node MUST 返回 `NWP-BUDGET-EXCEEDED` 错误
3. Node MUST NOT 静默截断结构化数据（截断可能导致 Agent 收到不完整结构）

---

## 5. CapsFrame 中的 token 估算

CapsFrame 的 `token_est` 字段值为 CGN：

```json
{
  "frame": "0x04",
  "anchor_ref": "sha256:...",
  "count": 2,
  "data": [...],
  "token_est": 180,
  "tokenizer_used": "cl100k_base"
}
```

---

## 6. 实现注意事项

### 6.1 通用

- Node 实现 SHOULD 内置至少 `cl100k_base`（GPT-4 系列）tokenizer。
- 汇率表建议作为可热更新配置，不硬编码。
- CGN 值始终为 uint32，最大 4,294,967,295。
- 在线传输的 CGN 值若不带显式 profile 标记，其默认 profile 为 **CGN-Estimate**。意图采用 CGN-Billing 语义的 Node MUST 按 §4.2 标记响应——沉默从来不是结算信号。

### 6.2 CGN-Estimate

- 高频场景下 token 估算 MAY 采样而非逐条计算。采样估值属于观察级信号，与 §3.2 警示对 `declared_tokenizer` 的限制相同：MUST NOT 单独用于计费、结算、配额提升、声誉或授权。
- 当 tokenizer 无法解析时，允许使用 §2.2 的字节数 fallback。
- 与原生计数之间允许 ±5 % 汇率漂移。
- CGN-Estimate 记录无需签名、审计日志持久化或争议处理机制。

### 6.3 CGN-Billing

- Node 在发出任何 CGN-Billing 记录之前，MUST 将该请求的 tokenizer 解析至 `verified_tokenizer` 等级（NPS-3-NIP §5.1）。若解析失败，Node MUST NOT 对该请求计费——见 §2.2。
- 禁止采样。每一个被计费的 CGN 值 MUST 使用 tokenizer 的原生计数与 pin 定的汇率表项进行逐条精确计算。
- 每条计量记录 MUST 由发起 Node 的 NID 签名，并 MUST 持久化到与 NOP §8.3（以及在已部署场景下 NPS-RFC-0004 logging）兼容的审计日志中。
- 审计日志 MUST 仅凭自身即可重建该笔收费、支撑争议 / 冲销处理，并能在 Node 进程重启后存活。
- 每一条 CGN-Billing 响应 MUST 同时携带 `X-NWP-Tokens-Profile: billing`、`X-NWP-Billing-Record` 与 `X-NWP-Billing-Tokenizer-Tier: verified_tokenizer` 三个响应头（详见 §4.2）。
- 覆盖 CGN-Billing 记录签名与验证的合规性测试向量将随下一版 AaaS-Profile L3 合规性套件一同发布（跟踪 issue #40）；在该套件正式发布之前，结算级实现 SHOULD 自行公开其签名方案，使对端可独立验证其声明。

---

## 7. 流式与订阅预算策略

`X-NWP-Budget` 上限仅适用于**同步请求/响应操作**（QueryFrame → CapsFrame / StreamFrame 批次）。以下持续推送操作遵循不同规则：

### 7.1 流式查询（QueryFrame `stream: true`）

- `X-NWP-Budget` 按**每个 StreamFrame 批次**执行，不针对整个流。
- 若处理某批次会超出声明预算，节点 MUST 对该批次进行裁剪或终止。
- 响应头 `X-NWP-Tokens` 仅报告当前批次消耗的 CGN。
- Agent 可在累计预算耗尽后主动断开连接。

### 7.2 SubscribeFrame / 推送流（topology.stream、事件订阅）

长连续推送流（如通过 SubscribeFrame 建立的 `topology.stream`）由一系列大小不固定的事件组成，预算语义有所不同：

| 方面 | 行为 |
|------|------|
| `X-NWP-Budget` 强制 | 节点不执行；推送事件的生成独立于任何请求级预算上限 |
| `X-NWP-Tokens` 上报 | 节点 SHOULD 在每个推送事件（DiffFrame）中附带此响应头，报告该事件有效载荷的 CGN |
| Agent 侧强制 | Agent 负责跨事件累计 CGN，并在会话预算耗尽时主动断开连接 |

> **设计原因**：对推送流执行 `X-NWP-Budget` 将要求节点缓冲未来事件，与实时拓扑变更交付的设计目标不兼容。订阅流的预算控制应由 Agent 侧负责。

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
