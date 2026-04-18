English | [中文版](./error-codes.cn.md)

# NPS Unified Error Code Namespace

**Version**: 0.4  
**Date**: 2026-04-14  

Error code format: `{PROTOCOL}-{CATEGORY}-{DETAIL}`

NPS uses a two-level error system:
1. **Protocol error codes** (this document): specific error identifiers prefixed with the owning protocol
2. **NPS status codes**: transport-layer status classification + HTTP mapping — see [status-codes.md](status-codes.md)

---

## NCP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NCP-ANCHOR-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | Schema referenced by `anchor_ref` does not exist; agent must re-fetch the AnchorFrame |
| `NCP-ANCHOR-SCHEMA-INVALID` | `NPS-CLIENT-BAD-FRAME` | Schema in AnchorFrame is malformed |
| `NCP-ANCHOR-ID-MISMATCH` | `NPS-CLIENT-CONFLICT` | Different Schemas received for the same `anchor_id` (anchor-pollution defense) |
| `NCP-FRAME-UNKNOWN-TYPE` | `NPS-CLIENT-BAD-FRAME` | Unknown frame type code |
| `NCP-FRAME-PAYLOAD-TOO-LARGE` | `NPS-LIMIT-PAYLOAD` | Payload exceeds the negotiated `max_frame_payload` |
| `NCP-FRAME-FLAGS-INVALID` | `NPS-CLIENT-BAD-FRAME` | Reserved bits in the flags field are non-zero |
| `NCP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | StreamFrame sequence numbers are not contiguous |
| `NCP-STREAM-NOT-FOUND` | `NPS-STREAM-NOT-FOUND` | The stream referenced by `stream_id` does not exist |
| `NCP-STREAM-LIMIT-EXCEEDED` | `NPS-STREAM-LIMIT` | Maximum concurrent streams per connection exceeded |
| `NCP-ENCODING-UNSUPPORTED` | `NPS-SERVER-ENCODING-UNSUPPORTED` | The requested encoding tier is not supported |
| `NCP-ANCHOR-STALE` | `NPS-CLIENT-CONFLICT` | `anchor_ref` exists but the schema has been updated; the response carries the latest AnchorFrame via `CapsFrame.inline_anchor` |
| `NCP-DIFF-FORMAT-UNSUPPORTED` | `NPS-CLIENT-BAD-FRAME` | DiffFrame uses a `patch_format` the receiver does not support (e.g., `binary_bitset`) |
| `NCP-VERSION-INCOMPATIBLE` | `NPS-PROTO-VERSION-INCOMPATIBLE` | Client `min_version` is higher than the server's maximum supported version (handshake failure) |
| `NCP-STREAM-WINDOW-OVERFLOW` | `NPS-STREAM-LIMIT` | Sender continues to emit StreamFrames after the application-layer flow-control window is exhausted |
| `NCP-ENC-NOT-NEGOTIATED` | `NPS-CLIENT-BAD-FRAME` | Received an ENC=1 frame but no E2E encryption algorithm was negotiated for the session (not declared in HelloFrame) |
| `NCP-ENC-AUTH-FAILED` | `NPS-CLIENT-BAD-FRAME` | E2E encryption auth-tag verification failed; the frame may have been tampered with |

---

## NWP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NWP-AUTH-NID-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Agent scope does not cover the target node path |
| `NWP-AUTH-NID-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | NID certificate has expired |
| `NWP-AUTH-NID-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | NID has been revoked |
| `NWP-AUTH-NID-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | NID issuer is not in `trusted_issuers` |
| `NWP-AUTH-NID-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Agent is missing a capability required by the node (e.g., `nwp:query`) |
| `NWP-QUERY-FILTER-INVALID` | `NPS-CLIENT-BAD-PARAM` | Filter syntax is invalid or nests deeper than 8 levels |
| `NWP-QUERY-FIELD-UNKNOWN` | `NPS-CLIENT-BAD-PARAM` | `fields` references an unknown field |
| `NWP-QUERY-CURSOR-INVALID` | `NPS-CLIENT-BAD-PARAM` | Cursor cannot be decoded or has expired |
| `NWP-QUERY-REGEX-UNSAFE` | `NPS-CLIENT-BAD-PARAM` | `$regex` pattern rejected (ReDoS risk, excessive length, or nested quantifiers) |
| `NWP-QUERY-VECTOR-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support vector search |
| `NWP-QUERY-AGGREGATE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support aggregate queries (`capabilities.aggregate=false`) |
| `NWP-QUERY-AGGREGATE-INVALID` | `NPS-CLIENT-BAD-PARAM` | Aggregate structure is invalid (unknown `func`, duplicate alias, missing required fields, etc.) |
| `NWP-QUERY-STREAM-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support streaming queries (`capabilities.stream_query=false`) |
| `NWP-ACTION-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `action_id` is not registered on the node |
| `NWP-ACTION-PARAMS-INVALID` | `NPS-CLIENT-UNPROCESSABLE` | Action params fail schema validation |
| `NWP-ACTION-IDEMPOTENCY-CONFLICT` | `NPS-CLIENT-CONFLICT` | A request with the same `idempotency_key` is already in progress |
| `NWP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | Asynchronous task referenced by `task_id` does not exist |
| `NWP-TASK-ALREADY-CANCELLED` | `NPS-CLIENT-CONFLICT` | Task has been cancelled; further operations are not permitted |
| `NWP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | Task has completed and cannot be cancelled |
| `NWP-TASK-ALREADY-FAILED` | `NPS-CLIENT-CONFLICT` | Task has failed and cannot be cancelled |
| `NWP-SUBSCRIBE-STREAM-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `stream_id` referenced by `unsubscribe` does not exist |
| `NWP-SUBSCRIBE-LIMIT-EXCEEDED` | `NPS-LIMIT-EXCEEDED` | Exceeded the node's maximum concurrent subscriptions |
| `NWP-SUBSCRIBE-FILTER-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node does not support subscriptions with a filter |
| `NWP-SUBSCRIBE-INTERRUPTED` | `NPS-SERVER-UNAVAILABLE` | Subscription stream terminated because the underlying data source was interrupted |
| `NWP-SUBSCRIBE-SEQ-TOO-OLD` | `NPS-CLIENT-CONFLICT` | `resume_from_seq` is outside the node's buffer window (recommended: 10 min or 10,000 records); agent must re-query from scratch before re-subscribing |
| `NWP-BUDGET-EXCEEDED` | `NPS-LIMIT-BUDGET` | Response would exceed the `X-NWP-Budget` limit |
| `NWP-DEPTH-EXCEEDED` | `NPS-CLIENT-BAD-PARAM` | `X-NWP-Depth` exceeds the node's permitted `max_depth` |
| `NWP-GRAPH-CYCLE` | `NPS-CLIENT-UNPROCESSABLE` | Node graph contains a cyclic reference |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | Underlying data source is temporarily unavailable |
| `NWP-MANIFEST-VERSION-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Agent's NPS version is lower than the node's `min_agent_version` |
| `NWP-RATE-LIMIT-EXCEEDED` | `NPS-LIMIT-RATE` | Rate limit exceeded; reset timestamp is in the `X-NWP-Rate-Reset` header |

---

## NIP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NIP-CERT-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | Certificate has expired (`expires_at < now`) |
| `NIP-CERT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | Certificate has been revoked (found in CRL or OCSP) |
| `NIP-CERT-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | Certificate signature verification failed |
| `NIP-CERT-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | Issuer is not in `trusted_issuers` |
| `NIP-CERT-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Certificate is missing a capability required by the node |
| `NIP-CERT-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Certificate scope does not cover the target path or operation |
| `NIP-CA-NID-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | NID does not exist in the CA database |
| `NIP-CA-NID-ALREADY-EXISTS` | `NPS-CLIENT-CONFLICT` | NID already exists (duplicate registration) |
| `NIP-CA-SERIAL-DUPLICATE` | `NPS-CLIENT-CONFLICT` | Certificate serial number already in use |
| `NIP-CA-RENEWAL-TOO-EARLY` | `NPS-CLIENT-BAD-PARAM` | More than 7 days until expiry; renewal window not yet open |
| `NIP-CA-SCOPE-EXPANSION-DENIED` | `NPS-AUTH-FORBIDDEN` | Requested scope exceeds the parent scope (delegation-chain violation) |
| `NIP-OCSP-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | OCSP service temporarily unavailable |
| `NIP-TRUST-FRAME-INVALID` | `NPS-CLIENT-BAD-FRAME` | TrustFrame signature or format is invalid |

---

## NDP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `nwp://` address cannot be resolved to a physical endpoint |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | Resolution result is conflicting (multiple inconsistent registrations) |
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Resolution request timed out |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame signature verification failed |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | NID in AnnounceFrame does not match the signing certificate |
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame sequence numbers are not contiguous |
| `NDP-REGISTRY-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | NDP Registry temporarily unavailable |

---

## NOP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NOP-TASK-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `task_id` does not exist |
| `NOP-TASK-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Overall task execution timed out |
| `NOP-TASK-DAG-INVALID` | `NPS-CLIENT-BAD-FRAME` | DAG structure is invalid (missing entry/exit node, field error, etc.) |
| `NOP-TASK-DAG-CYCLE` | `NPS-CLIENT-BAD-FRAME` | DAG contains a cycle |
| `NOP-TASK-DAG-TOO-LARGE` | `NPS-CLIENT-BAD-FRAME` | DAG node count exceeds the limit (default 32) |
| `NOP-TASK-ALREADY-COMPLETED` | `NPS-CLIENT-CONFLICT` | Task has completed; cannot be resubmitted |
| `NOP-TASK-CANCELLED` | `NPS-CLIENT-CONFLICT` | Task has been cancelled |
| `NOP-DELEGATE-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | `delegated_scope` exceeds the parent agent's scope |
| `NOP-DELEGATE-REJECTED` | `NPS-CLIENT-UNPROCESSABLE` | Target agent declined the delegation (insufficient capability or overload); response includes `retry_after_ms` |
| `NOP-DELEGATE-CHAIN-TOO-DEEP` | `NPS-CLIENT-BAD-PARAM` | Delegation chain depth exceeds the limit (default 3 levels) |
| `NOP-DELEGATE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Subtask did not complete before `deadline_at` |
| `NOP-SYNC-TIMEOUT` | `NPS-SERVER-TIMEOUT` | SyncFrame timed out waiting for dependent tasks |
| `NOP-SYNC-DEPENDENCY-FAILED` | `NPS-CLIENT-UNPROCESSABLE` | Dependency subtask has failed (and failure count exceeds the K-of-N tolerance) |
| `NOP-STREAM-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | AlignStream sequence numbers are not contiguous |
| `NOP-STREAM-NID-MISMATCH` | `NPS-AUTH-UNAUTHENTICATED` | AlignStream `sender_nid` does not match the connection identity |
| `NOP-RESOURCE-INSUFFICIENT` | `NPS-SERVER-UNAVAILABLE` | Preflight found one or more Worker Agents lack sufficient resources (NPT or capabilities) |
| `NOP-CONDITION-EVAL-ERROR` | `NPS-CLIENT-BAD-PARAM` | DAG node `condition` expression failed to evaluate (syntax error or missing referenced field) |
| `NOP-INPUT-MAPPING-ERROR` | `NPS-CLIENT-UNPROCESSABLE` | `input_mapping` JSONPath could not be resolved or target field is missing |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
