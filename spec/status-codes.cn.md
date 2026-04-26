[English Version](./status-codes.md) | 中文版

# NPS 原生状态码与 HTTP 映射

**Version**: 0.4  
**Date**: 2026-04-26  

NPS 定义独立于 HTTP 的原生状态码体系。原生模式下使用 NPS 状态码；HTTP/Overlay 模式下额外提供 HTTP 状态码映射。

---

## 状态码格式

```
{PROTOCOL}-{CATEGORY}-{DETAIL}
```

全大写，连字符分隔。错误码的完整列表见 [error-codes.cn.md](error-codes.cn.md)。

---

## NPS 原生状态分类

| 分类码 | 含义 | 类比 |
|--------|------|------|
| `OK` | 操作成功 | HTTP 2xx |
| `CLIENT` | 客户端错误（请求格式、参数、权限） | HTTP 4xx |
| `SERVER` | 服务端错误（内部故障、暂不可用） | HTTP 5xx |
| `STREAM` | 流式传输相关错误 | — |
| `AUTH` | 身份认证与授权错误 | HTTP 401/403 |
| `LIMIT` | 资源或配额限制 | HTTP 429 |
| `PROTO` | 协议层、握手前错误（版本不匹配、连接前导格式错误）；并不总在线上发出 | — |

---

## NPS 状态码 → HTTP 映射表

> 此映射仅在 HTTP/Overlay 模式下使用。原生模式下，错误通过 NCP 帧直接携带状态码字符串。

### 成功

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-OK` | 200 | 操作成功 |
| `NPS-OK-ACCEPTED` | 202 | 异步操作已接受 |
| `NPS-OK-NO-CONTENT` | 204 | 操作成功，无响应体 |

### 客户端错误（CLIENT）

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-CLIENT-BAD-FRAME` | 400 | 帧格式不合法 |
| `NPS-CLIENT-BAD-PARAM` | 400 | 请求参数不合法 |
| `NPS-CLIENT-NOT-FOUND` | 404 | 目标资源不存在 |
| `NPS-CLIENT-CONFLICT` | 409 | 资源状态冲突 |
| `NPS-CLIENT-GONE` | 410 | 资源已永久移除 |
| `NPS-CLIENT-UNPROCESSABLE` | 422 | 请求语义错误 |

### 认证与授权（AUTH）

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-AUTH-UNAUTHENTICATED` | 401 | 未提供身份凭证或凭证无效 |
| `NPS-AUTH-FORBIDDEN` | 403 | 身份有效但权限不足 |

### 资源限制（LIMIT）

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-LIMIT-RATE` | 429 | 请求频率超限 |
| `NPS-LIMIT-BUDGET` | 429 | Token 预算超限 |
| `NPS-LIMIT-PAYLOAD` | 413 | Payload 超过帧大小上限 |

### 服务端错误（SERVER）

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-SERVER-INTERNAL` | 500 | 内部错误 |
| `NPS-SERVER-UNAVAILABLE` | 503 | 服务暂不可用 |
| `NPS-SERVER-TIMEOUT` | 408/504 | 操作超时 |
| `NPS-SERVER-ENCODING-UNSUPPORTED` | 415 | 不支持请求的编码格式 |
| `NPS-DOWNSTREAM-UNAVAILABLE` | 502/503 | 节点依赖的下游服务（声誉日志运营方、外部认证服务等）不可达；与 `NPS-SERVER-UNAVAILABLE` 区分，让客户端在策略允许时可重试备选下游（NPS-RFC-0004）|

### 流式（STREAM）

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-STREAM-SEQ-GAP` | 422 | 序号不连续 |
| `NPS-STREAM-NOT-FOUND` | 404 | 流 ID 不存在 |
| `NPS-STREAM-LIMIT` | 429 | 超出并发流上限 |

### 协议层（PROTO）

> 握手前或传输边界错误。这类错误在帧解析器运行之前就被识别（或代替帧解析）。在原生模式下，通常**不**通过 ErrorFrame 返回 —— 因为对端尚未确认是否说 NCP，连接被静默关闭。状态码本身仍然存在，用于 SDK 内部 telemetry（日志、metrics、关闭原因分类）。

| NPS 状态码 | HTTP 状态 | 描述 |
|-----------|-----------|------|
| `NPS-PROTO-VERSION-INCOMPATIBLE` | 426 | 客户端 `min_version` 高于服务端最高支持版本（HelloFrame 协商失败）。HTTP 模式返回 426 Upgrade Required；原生模式发一个 ErrorFrame 后断开。 |
| `NPS-PROTO-PREAMBLE-INVALID` | 400（不发出） | 原生模式连接首 8 字节非常量前导 `b"NPS/1.0\n"`。服务端 500 ms 内静默关闭，不发 ErrorFrame。（NPS-RFC-0001） |

---

## 协议错误码到 NPS 状态码的映射

各协议的具体错误码（如 `NCP-ANCHOR-NOT-FOUND`）映射到对应的 NPS 状态分类：

| 协议错误码 | NPS 状态码 |
|-----------|-----------|
| `NCP-ANCHOR-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` |
| `NCP-ANCHOR-SCHEMA-INVALID` | `NPS-CLIENT-BAD-FRAME` |
| `NCP-ANCHOR-ID-MISMATCH` | `NPS-CLIENT-CONFLICT` |
| `NCP-FRAME-UNKNOWN-TYPE` | `NPS-CLIENT-BAD-FRAME` |
| `NCP-FRAME-PAYLOAD-TOO-LARGE` | `NPS-LIMIT-PAYLOAD` |
| `NCP-FRAME-FLAGS-INVALID` | `NPS-CLIENT-BAD-FRAME` |
| `NCP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` |
| `NCP-STREAM-NOT-FOUND` | `NPS-STREAM-NOT-FOUND` |
| `NCP-STREAM-LIMIT-EXCEEDED` | `NPS-STREAM-LIMIT` |
| `NCP-ENCODING-UNSUPPORTED` | `NPS-SERVER-ENCODING-UNSUPPORTED` |
| `NCP-VERSION-INCOMPATIBLE` | `NPS-PROTO-VERSION-INCOMPATIBLE` |
| `NCP-PREAMBLE-INVALID` | `NPS-PROTO-PREAMBLE-INVALID` |
| `NIP-ASSURANCE-MISMATCH` | `NPS-CLIENT-BAD-FRAME` |
| `NIP-ASSURANCE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` |
| `NIP-REPUTATION-ENTRY-INVALID` | `NPS-CLIENT-BAD-FRAME` |
| `NIP-REPUTATION-LOG-UNREACHABLE` | `NPS-DOWNSTREAM-UNAVAILABLE` |
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` |
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` |
| `NWP-AUTH-NID-*` | `NPS-AUTH-*`（按具体错误映射）|
| `NWP-BUDGET-EXCEEDED` | `NPS-LIMIT-BUDGET` |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` |
| `NIP-CERT-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` |
| `NIP-CERT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` |
| `NIP-CERT-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` |
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` |
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` |
| `NOP-TASK-TIMEOUT` | `NPS-SERVER-TIMEOUT` |
| `NOP-DELEGATE-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` |

完整映射见各子协议的错误码章节。

---

## 原生模式错误帧格式

在原生模式下，错误通过 NCP 错误帧返回：

```json
{
  "frame": "0xFE",
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NCP-ANCHOR-NOT-FOUND",
  "anchor_ref": "sha256:...",
  "message": "Schema anchor not found in cache, please resend AnchorFrame"
}
```

帧类型 `0xFE` 为 NPS 统一错误帧（从 Reserved 范围分配），所有协议层共用。

---

*归属：LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
