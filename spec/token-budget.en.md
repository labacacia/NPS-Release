English | [中文版](./token-budget.md)

# NPS Token Budget Specification

**Version**: 0.1  
**Date**: 2026-04-12  

---

## 1. Overview

The NPS Token Budget mechanism lets an agent declare a maximum token-consumption cap for a given request. A node uses this cap to trim response fields, limit the number of records returned, or reject over-budget requests.

To address the differences in how each LLM counts tokens, NPS introduces **NPS Token (NPT)** as a standardized unit of measure.

---

## 2. NPS Token (NPT)

### 2.1 Definition

NPT is the standard token-accounting unit inside the NPS protocol suite. Native tokens from each LLM are converted to NPT through exchange rates.

### 2.2 Default Calculation (Fallback)

When the tokenizer cannot be determined, use the following formula as a default estimate:

```
NPT = ceil(UTF-8_bytes / 4)
```

This formula reflects the average behavior of mainstream LLM tokenizers (≈ 4 bytes/token for English, ≈ 3 bytes/token for Chinese) and acts as the most conservative baseline.

### 2.3 Exchange Rate Table (NPT Exchange Rates)

Node implementations SHOULD ship with built-in NPT exchange rates for common models:

| Model family | Tokenizer | 1 native token ≈ NPT | Notes |
|--------------|-----------|-----------------------|-------|
| OpenAI GPT-4 / GPT-4o | `cl100k_base` | 1.0 | Reference baseline |
| Anthropic Claude | Claude tokenizer | 1.05 | Slightly higher than GPT-4 |
| Google Gemini | SentencePiece | 0.95 | Slightly lower than GPT-4 |
| Meta LLaMA 3 | `llama3-tokenizer` | 1.02 | Near baseline |
| Mistral | SentencePiece | 0.98 | Near baseline |
| Default (unknown model) | UTF-8 / 4 | 1.0 | Fallback |

The table is maintained with NPS version updates. Implementations MAY override the built-in rates via configuration.

---

## 3. Tokenizer Resolution Chain

When an agent issues a request, the node resolves the tokenizer in this order:

```
1. Explicit declaration by the agent (X-NWP-Tokenizer header)
   ↓ not declared
2. Auto-match from agent configuration / IdentFrame
   ↓ match failed
3. Default calculation (UTF-8 bytes / 4)
```

### 3.1 Explicit Declaration (highest priority)

The agent declares its tokenizer in the request header:

```
X-NWP-Tokenizer: cl100k_base
```

Node MUST recognize the declared tokenizer and use the corresponding algorithm to count tokens. If the node does not support that tokenizer, it SHOULD fall back to auto-match.

### 3.2 Auto-Match

The node infers the agent's model family from IdentFrame metadata:

- `IdentFrame.metadata.model_family`: e.g. `"openai/gpt-4o"`, `"anthropic/claude-4"`
- `IdentFrame.metadata.tokenizer`: e.g. `"cl100k_base"`

When either field is present in the IdentFrame, the node uses the matching tokenizer.

### 3.3 Default Fallback

When the tokenizer cannot be determined, use `ceil(UTF-8_bytes / 4)` to compute NPT.

---

## 4. Request & Response

### 4.1 Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-NWP-Budget` | optional | Maximum NPT budget (uint32) |
| `X-NWP-Tokenizer` | optional | Tokenizer identifier used by the agent |

### 4.2 Response Headers

| Header | Description |
|--------|-------------|
| `X-NWP-Tokens` | Actual NPT consumed by this response |
| `X-NWP-Tokens-Native` | Native token consumption for this response (when the tokenizer is known) |
| `X-NWP-Tokenizer-Used` | Tokenizer identifier actually used by the node |

### 4.3 Over-Budget Handling

When the response would exceed `X-NWP-Budget`:

1. Node SHOULD trim the response first (fewer fields or records) to fit within budget.
2. If trimming is impossible (e.g. a single record already exceeds budget), node MUST return a `NWP-BUDGET-EXCEEDED` error.
3. Node MUST NOT silently truncate structured data (truncation can produce incomplete structures on the agent side).

---

## 5. Token Estimate in CapsFrame

The `token_est` field in a CapsFrame is in NPT:

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

## 6. Implementation Notes

- Node implementations SHOULD ship with at least the `cl100k_base` (GPT-4 family) tokenizer built in.
- The exchange-rate table should be hot-reloadable configuration, not hard-coded.
- For high-frequency scenarios, token estimation MAY be sampled rather than computed record-by-record.
- NPT values are always uint32 — maximum 4,294,967,295.

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
