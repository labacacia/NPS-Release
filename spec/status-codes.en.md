English | [中文版](./status-codes.md)

# NPS Native Status Codes and HTTP Mapping

**Version**: 0.1  
**Date**: 2026-04-12  

NPS defines a native status-code system independent of HTTP. Native mode uses NPS status codes directly; HTTP / Overlay mode additionally provides a mapping to HTTP status codes.

---

## Status Code Format

```
{PROTOCOL}-{CATEGORY}-{DETAIL}
```

All uppercase, hyphen-separated. For the full error-code list, see [error-codes.en.md](error-codes.en.md).

---

## NPS Native Status Categories

| Category | Meaning | Analogue |
|----------|---------|----------|
| `OK` | Successful operation | HTTP 2xx |
| `CLIENT` | Client-side error (format, params, permission) | HTTP 4xx |
| `SERVER` | Server-side error (internal fault, unavailable) | HTTP 5xx |
| `STREAM` | Stream transport-related errors | — |
| `AUTH` | Authentication / authorization errors | HTTP 401/403 |
| `LIMIT` | Resource or quota limits | HTTP 429 |

---

## NPS Status → HTTP Mapping

> This mapping is used only in HTTP / Overlay mode. In native mode, errors are carried directly as status-code strings inside NCP frames.

### Success

| NPS Status | HTTP | Description |
|------------|------|-------------|
| `NPS-OK` | 200 | Operation succeeded |
| `NPS-OK-ACCEPTED` | 202 | Asynchronous operation accepted |
| `NPS-OK-NO-CONTENT` | 204 | Operation succeeded with no response body |

### Client Errors (CLIENT)

| NPS Status | HTTP | Description |
|------------|------|-------------|
| `NPS-CLIENT-BAD-FRAME` | 400 | Frame format invalid |
| `NPS-CLIENT-BAD-PARAM` | 400 | Request parameter invalid |
| `NPS-CLIENT-NOT-FOUND` | 404 | Target resource does not exist |
| `NPS-CLIENT-CONFLICT` | 409 | Resource state conflict |
| `NPS-CLIENT-GONE` | 410 | Resource permanently removed |
| `NPS-CLIENT-UNPROCESSABLE` | 422 | Request semantics invalid |

### Authentication & Authorization (AUTH)

| NPS Status | HTTP | Description |
|------------|------|-------------|
| `NPS-AUTH-UNAUTHENTICATED` | 401 | No identity credential provided or credential invalid |
| `NPS-AUTH-FORBIDDEN` | 403 | Identity is valid but lacks permission |

### Resource Limits (LIMIT)

| NPS Status | HTTP | Description |
|------------|------|-------------|
| `NPS-LIMIT-RATE` | 429 | Request rate exceeded |
| `NPS-LIMIT-BUDGET` | 429 | Token budget exceeded |
| `NPS-LIMIT-PAYLOAD` | 413 | Payload exceeds maximum frame size |

### Server Errors (SERVER)

| NPS Status | HTTP | Description |
|------------|------|-------------|
| `NPS-SERVER-INTERNAL` | 500 | Internal error |
| `NPS-SERVER-UNAVAILABLE` | 503 | Service temporarily unavailable |
| `NPS-SERVER-TIMEOUT` | 408/504 | Operation timed out |
| `NPS-SERVER-ENCODING-UNSUPPORTED` | 415 | Requested encoding tier is not supported |

### Stream (STREAM)

| NPS Status | HTTP | Description |
|------------|------|-------------|
| `NPS-STREAM-SEQ-GAP` | 422 | Sequence gap |
| `NPS-STREAM-NOT-FOUND` | 404 | Stream id does not exist |
| `NPS-STREAM-LIMIT` | 429 | Concurrent stream limit exceeded |

---

## Protocol-Error → NPS-Status Mapping

Each protocol's specific error code (e.g. `NCP-ANCHOR-NOT-FOUND`) maps to the corresponding NPS status category:

| Protocol Error | NPS Status |
|----------------|-----------|
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
| `NWP-AUTH-NID-*` | `NPS-AUTH-*` (maps per specific error) |
| `NWP-BUDGET-EXCEEDED` | `NPS-LIMIT-BUDGET` |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` |
| `NIP-CERT-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` |
| `NIP-CERT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` |
| `NIP-CERT-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` |
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` |
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` |
| `NOP-TASK-TIMEOUT` | `NPS-SERVER-TIMEOUT` |
| `NOP-DELEGATE-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` |

For the complete mapping, see each sub-protocol's error-code chapter.

---

## Native-Mode Error Frame Format

In native mode, errors are returned via an NCP error frame:

```json
{
  "frame": "0xFE",
  "status": "NPS-CLIENT-NOT-FOUND",
  "error": "NCP-ANCHOR-NOT-FOUND",
  "anchor_ref": "sha256:...",
  "message": "Schema anchor not found in cache, please resend AnchorFrame"
}
```

Frame type `0xFE` is the NPS unified error frame (allocated from the Reserved range), shared across all protocol layers.

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
