English | [中文版](./token-budget.cn.md)

# Cognon Budget Specification

**Version**: 0.5
**Date**: 2026-05-09

---

## 1. Overview

The Cognon Budget mechanism lets an agent declare a maximum token-consumption cap for a given request. A node uses this cap to trim response fields, limit the number of records returned, or reject over-budget requests.

To address the differences in how each LLM counts tokens, NPS introduces **Cognon (CGN)** as a standardized unit of measure.

---

## 2. Cognon (CGN)

### 2.1 Definition

CGN is the standard token-accounting unit inside the NPS protocol suite. Native tokens from each LLM are converted to CGN through exchange rates.

CGN is defined in **two named profiles** with non-overlapping conformance requirements (issue #40). Every CGN value carried on the wire MUST be unambiguously associated with exactly one profile; counterparties MUST NOT mix the two.

| Profile | Purpose | Used by |
|---------|---------|---------|
| **CGN-Estimate** | Estimation, budget hints, telemetry, sampling-tolerant flows | `X-NWP-Budget` enforcement, CapsFrame `token_est`, push-stream per-event `cgn_est` reporting |
| **CGN-Billing** | Commercial settlement, dispute and chargeback handling | Invoiced metering and signed accounting records exchanged between counterparties |

#### CGN-Estimate (estimation-grade)

- Tokenizer source MAY be `declared_tokenizer` (NPS-3-NIP §5.1) or any tier above.
- The byte-size fallback in §2.2 is permitted.
- Sampling is permitted (see §6.2).
- Exchange-rate drift up to ±5 % against the model-native count is acceptable.
- Records are unsigned; no audit-log integration is required.

#### CGN-Billing (settlement-grade)

A node that emits CGN-Billing records MUST satisfy **all** of the following:

- Tokenizer used MUST be `verified_tokenizer` tier per [NPS-3-NIP §5.1](./NPS-3-NIP.md#trust-boundary-for-unsigned-metadata-normative--closes-issue-39). `declared_tokenizer`, `observed_tokenizer_profile`, and the §2.2 byte-size fallback are **forbidden** as billing inputs.
- Each metering record MUST be signed by the issuing node's NID and MUST be persisted in an audit log compatible with NOP §8.3 (and where deployed, NPS-RFC-0004 logging).
- Sampling MUST NOT be used. Every billed CGN value MUST be computed exactly, record-by-record.
- Counts MUST be deterministic and non-approximated; the ±5 % drift envelope of §2.3 does NOT apply, because billing rates are an exact contract term, not an estimate.
- Disputes and chargebacks MUST be supportable from the audit log alone, without reference to ephemeral node state.
- Exchange-rate table version MUST be pinned by both counterparties at session-start time (or earlier) and recorded inside the signed metering record.

A node that issues a CGN-denominated commercial charge MUST mark it as CGN-Billing on the wire and in headers (see §4.2). Charges presented as CGN-Estimate, or with no profile marker, are non-conformant for settlement and MAY be disputed by the counterparty without reference to a tokenizer trust tier.

### 2.2 Default Calculation (Fallback) — CGN-Estimate only

When the tokenizer cannot be determined, **CGN-Estimate** MAY fall back to:

```
CGN = ceil(UTF-8_bytes / 4)
```

This formula reflects the average behavior of mainstream LLM tokenizers (≈ 4 bytes/token for English, ≈ 3 bytes/token for Chinese) and acts as the most conservative baseline.

The byte-size fallback MUST NOT be used for **CGN-Billing** under any circumstances. If a node cannot resolve a `verified_tokenizer` for a request that would be billed, it MUST refuse to issue a CGN-Billing record for that request and either (a) downgrade the surface to CGN-Estimate (non-billable telemetry only) or (b) reject the request with a billing-class error.

### 2.3 Exchange Rate Table (CGN Exchange Rates)

Node implementations SHOULD ship with built-in CGN exchange rates for common models:

| Model family | Tokenizer | 1 native token ≈ CGN | Notes |
|--------------|-----------|-----------------------|-------|
| OpenAI GPT-4 / GPT-4o | `cl100k_base` | 1.0 | Reference baseline |
| Anthropic Claude | Claude tokenizer | 1.05 | Slightly higher than GPT-4 |
| Google Gemini | SentencePiece | 0.95 | Slightly lower than GPT-4 |
| Meta LLaMA 3 | `llama3-tokenizer` | 1.02 | Near baseline |
| Mistral | SentencePiece | 0.98 | Near baseline |
| Default (unknown model) | UTF-8 / 4 | 1.0 | Fallback (CGN-Estimate only) |

The table is maintained with NPS version updates. Implementations MAY override the built-in rates via configuration.

**Profile applicability.** The rates above are normative for **CGN-Estimate** and tolerate the documented ±5 % drift between table values and the model's native count. For **CGN-Billing**, both counterparties MUST agree on a specific table version (pinned at session-start time or earlier and recorded inside the signed metering record) and MUST use the matching `verified_tokenizer`-derived native count; the ±5 % envelope does NOT apply. The `Default (unknown model)` row applies only to CGN-Estimate; CGN-Billing has no fallback row.

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

> **Estimation-only caveat (normative — issue #39).** Both `metadata.model_family` and `metadata.tokenizer` reach the node as **`declared_tokenizer`** under the three-tier tokenizer trust model defined in [NPS-3-NIP §5.1 — Trust boundary for unsigned `metadata`](./NPS-3-NIP.md#trust-boundary-for-unsigned-metadata-normative--closes-issue-39). The `X-NWP-Tokenizer` request header in §3.1 carries the same trust class. Auto-matched values from this section MUST be treated as **estimation hints only** and MUST NOT drive billing, settlement, quota elevation, reputation scoring, or any security-relevant decision. Settlement-grade and policy-grade flows MUST instead consume a `verified_tokenizer` (CA- or platform-attested) signal, falling back to `observed_tokenizer_profile` only for Node-internal abuse detection. A Node that bills or grants elevated quota off a `declared_tokenizer` is non-conformant.

### 3.3 Default Fallback

When the tokenizer cannot be determined, use `ceil(UTF-8_bytes / 4)` to compute CGN.

---

## 4. Request & Response

### 4.1 Request Headers

| Header | Required | Description |
|--------|----------|-------------|
| `X-NWP-Budget` | optional | Maximum CGN budget (uint32) |
| `X-NWP-Tokenizer` | optional | Tokenizer identifier used by the agent |

### 4.2 Response Headers

| Header | Profile | Description |
|--------|---------|-------------|
| `X-NWP-Tokens` | CGN-Estimate | Actual CGN consumed by this response (estimation-grade) |
| `X-NWP-Tokens-Native` | CGN-Estimate | Native token consumption for this response (when the tokenizer is known) |
| `X-NWP-Tokenizer-Used` | both | Tokenizer identifier actually used by the node |
| `X-NWP-Tokens-Profile` | both | Either `estimate` or `billing`. Absent or `estimate` MUST be treated as CGN-Estimate by the counterparty. |
| `X-NWP-Billing-Record` | CGN-Billing | Reference (URI or content-hash) to the signed metering record for this response. MUST be present iff the response is billed under CGN-Billing. |
| `X-NWP-Billing-Tokenizer-Tier` | CGN-Billing | MUST be `verified_tokenizer`. Absent → not billable. |

A response that omits both `X-NWP-Billing-Record` and `X-NWP-Billing-Tokenizer-Tier` MUST be interpreted by the counterparty as CGN-Estimate, regardless of any commercial agreement; nodes MUST NOT settle off CGN-Estimate-only responses.

### 4.3 Over-Budget Handling

When the response would exceed `X-NWP-Budget`:

1. Node SHOULD trim the response first (fewer fields or records) to fit within budget.
2. If trimming is impossible (e.g. a single record already exceeds budget), node MUST return a `NWP-BUDGET-EXCEEDED` error.
3. Node MUST NOT silently truncate structured data (truncation can produce incomplete structures on the agent side).

---

## 5. Token Estimate in CapsFrame

The `token_est` field in a CapsFrame is in CGN:

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

### 6.1 General

- Node implementations SHOULD ship with at least the `cl100k_base` (GPT-4 family) tokenizer built in.
- The exchange-rate table should be hot-reloadable configuration, not hard-coded.
- CGN values are always uint32 — maximum 4,294,967,295.
- The default profile for any CGN value carried on the wire without an explicit profile marker is **CGN-Estimate**. A node intending CGN-Billing semantics MUST mark the response per §4.2 — silence is never a settlement signal.

### 6.2 CGN-Estimate

- For high-frequency scenarios, token estimation MAY be sampled rather than computed record-by-record. Sampled estimates are observation-grade signals and inherit the same restriction as `declared_tokenizer` (see §3.2 caveat): they MUST NOT be the sole basis for billing, settlement, quota elevation, reputation, or authorization.
- The byte-size fallback (§2.2) is permitted whenever the tokenizer cannot be resolved.
- Exchange-rate drift up to ±5 % against the native count is tolerated.
- CGN-Estimate records do not require signing, audit-log persistence, or dispute machinery.

### 6.3 CGN-Billing

- A node MUST resolve the request's tokenizer at the `verified_tokenizer` tier (NPS-3-NIP §5.1) before emitting any CGN-Billing record. If resolution fails, the node MUST NOT bill the request — see §2.2.
- Sampling is forbidden. Every billed CGN value MUST be computed exactly, per record, using the tokenizer's native count and the pinned exchange-rate table entry.
- Each metering record MUST be NID-signed by the issuing node and MUST be persisted in an audit log compatible with NOP §8.3 (and where deployed, NPS-RFC-0004 logging).
- The audit log MUST be sufficient on its own to reconstruct the charge, support dispute / chargeback handling, and survive node-process restarts.
- The `X-NWP-Tokens-Profile: billing`, `X-NWP-Billing-Record`, and `X-NWP-Billing-Tokenizer-Tier: verified_tokenizer` headers (§4.2) MUST all be present on every CGN-Billing response.
- Conformance vectors covering CGN-Billing record signing and verification will be published alongside the next AaaS-Profile L3 conformance suite (tracking issue #40); until those are ratified, billing-grade implementations SHOULD self-publish their signing scheme so counterparties can verify claims independently.

---

## 7. Streaming and Subscription Budget Policy

The `X-NWP-Budget` cap applies to **synchronous request/response operations** (QueryFrame → CapsFrame / StreamFrame batch). The following continuous-push operations are subject to modified rules:

### 7.1 Streaming Queries (QueryFrame `stream: true`)

- `X-NWP-Budget` applies **per StreamFrame batch**, not to the total stream.
- The node MUST trim or stop the batch if processing that batch would exceed the declared budget.
- `X-NWP-Tokens` in the response header reports the CGN consumed by the current batch only.
- The Agent MAY choose to disconnect early once its cumulative budget is exhausted.

### 7.2 SubscribeFrame / Push Streams (topology.stream, event subscriptions)

Long-running push streams (e.g. `topology.stream` via SubscribeFrame) represent an ongoing series of events with no fixed response size. Budget semantics differ:

| Aspect | Behavior |
|--------|----------|
| `X-NWP-Budget` enforcement | Not applied by the node; push events are generated independently of any per-request budget cap |
| `X-NWP-Tokens` reporting | The node SHOULD include this header on each push event (DiffFrame) reporting the CGN for that event's payload |
| Agent-side enforcement | The Agent is responsible for tracking cumulative CGN across events and disconnecting when its session budget is exhausted |

> **Rationale**: Enforcing `X-NWP-Budget` on push streams would require the node to buffer future events, which is incompatible with real-time topology change delivery. Agent-side enforcement is the correct locus for subscription-stream budget control.

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
