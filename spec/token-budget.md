[English Version](./token-budget.en.md) | 中文版

# NPS Token Budget 规范

**Version**: 0.1  
**Date**: 2026-04-12  

---

## 1. 概述

NPS Token Budget 机制允许 Agent 在请求中声明本次操作的最大 token 消耗上限。Node 据此裁剪响应字段、限制返回条数或拒绝超预算请求。

为解决不同 LLM 的 token 计算差异，NPS 引入 **NPS Token（NPT）** 作为标准化计量单位。

---

## 2. NPS Token（NPT）

### 2.1 定义

NPS Token（NPT）是 NPS 协议族内部的标准 token 计量单位。各 LLM 的原生 token 通过汇率转换为 NPT。

### 2.2 默认计算方法（Fallback）

当 tokenizer 无法确定时，使用以下公式作为默认估算：

```
NPT = ceil(UTF-8_bytes / 4)
```

此公式基于主流 LLM tokenizer 的平均行为（英文约 4 bytes/token，中文约 3 bytes/token），作为最保守的估算基线。

### 2.3 汇率表（NPT Exchange Rates）

Node 实现 SHOULD 内置常见模型的 NPT 汇率：

| 模型族 | Tokenizer | 1 原生 Token ≈ NPT | 说明 |
|--------|-----------|---------------------|------|
| OpenAI GPT-4 / GPT-4o | `cl100k_base` | 1.0 | 基准参照 |
| Anthropic Claude | Claude tokenizer | 1.05 | 略高于 GPT-4 |
| Google Gemini | SentencePiece | 0.95 | 略低于 GPT-4 |
| Meta LLaMA 3 | `llama3-tokenizer` | 1.02 | 接近基准 |
| Mistral | SentencePiece | 0.98 | 接近基准 |
| Default（未知模型）| UTF-8 / 4 | 1.0 | Fallback |

汇率表随 NPS 版本更新维护。实现 MAY 通过配置覆盖内置汇率。

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

### 3.3 默认 Fallback

无法确定 tokenizer 时，使用 `ceil(UTF-8_bytes / 4)` 计算 NPT。

---

## 4. 请求与响应

### 4.1 请求头

| 头 | 必填 | 描述 |
|----|------|------|
| `X-NWP-Budget` | 可选 | 最大 NPT 预算（uint32）|
| `X-NWP-Tokenizer` | 可选 | Agent 使用的 tokenizer 标识 |

### 4.2 响应头

| 头 | 描述 |
|----|------|
| `X-NWP-Tokens` | 本响应实际 NPT 消耗 |
| `X-NWP-Tokens-Native` | 本响应原生 token 消耗（若已知 tokenizer） |
| `X-NWP-Tokenizer-Used` | Node 实际使用的 tokenizer 标识 |

### 4.3 超预算处理

当响应将超过 `X-NWP-Budget` 时：

1. Node SHOULD 优先裁剪响应（减少返回字段或条数），使结果在预算内
2. 若无法裁剪（如单条记录已超预算），Node MUST 返回 `NWP-BUDGET-EXCEEDED` 错误
3. Node MUST NOT 静默截断结构化数据（截断可能导致 Agent 收到不完整结构）

---

## 5. CapsFrame 中的 token 估算

CapsFrame 的 `token_est` 字段值为 NPT：

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

- Node 实现 SHOULD 内置至少 `cl100k_base`（GPT-4 系列）tokenizer
- 汇率表建议作为可热更新配置，不硬编码
- 高频场景可对 token 估算做采样而非逐条计算
- NPT 值始终为 uint32，最大 4,294,967,295

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
