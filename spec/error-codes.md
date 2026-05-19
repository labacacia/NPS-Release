English | [中文版](./error-codes.cn.md)

# NPS Unified Error Code Namespace

**Version**: 1.4
**Date**: 2026-05-11

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
| `NCP-PREAMBLE-INVALID` | `NPS-PROTO-PREAMBLE-INVALID` | Native-mode connection opened with bytes other than the constant preamble `b"NPS/1.0\n"`; server closes the connection silently without emitting an ErrorFrame (NPS-RFC-0001) |

---

## NWP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NWP-AUTH-NID-SCOPE-VIOLATION` | `NPS-AUTH-FORBIDDEN` | Agent scope does not cover the target node path |
| `NWP-AUTH-NID-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | NID certificate has expired |
| `NWP-AUTH-NID-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | NID has been revoked |
| `NWP-AUTH-NID-UNTRUSTED-ISSUER` | `NPS-AUTH-UNAUTHENTICATED` | NID issuer is not in `trusted_issuers` |
| `NWP-AUTH-NID-CAPABILITY-MISSING` | `NPS-AUTH-FORBIDDEN` | Agent is missing a capability required by the node (e.g., `nwp:query`) |
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` | Agent's assurance level is below the node's `min_assurance_level` (NPS-2 §4.1) or per-action override (§4.6); response SHOULD include a `hint` pointing to a CA enrolment URL (NPS-RFC-0003) |
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` | **Deprecated** (RFC-0005): use `NWP-REPUTATION-REJECTED` / `NWP-REPUTATION-BANNED` instead. Retained as alias for one alpha cycle. |
| `NWP-REPUTATION-THROTTLED` | `NPS-CLIENT-RATE-LIMITED` | Request rate-limited by reputation policy (`throttle_on` rule matched). Response includes `Retry-After: 60` header (NPS-RFC-0005) |
| `NWP-REPUTATION-REJECTED` | `NPS-AUTH-FORBIDDEN` | Request rejected by reputation policy (`reject_on` rule matched). Response body includes `matched_incident` + `matched_severity` (NPS-RFC-0005) |
| `NWP-REPUTATION-BANNED` | `NPS-AUTH-FORBIDDEN` | Request rejected and NID temporarily banned (`ban_on` rule matched or active ban cache entry). Response SHOULD include `X-NWP-Ban-Expires` Unix timestamp (NPS-RFC-0005) |
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
| `NWP-CGN-LIMIT-EXCEEDED` | `NPS-CLIENT-REQUEST-TOO-LARGE` | Response would exceed the effective CGN budget (`min(cgn_limit, X-NWP-Budget)`); trimming was not possible. Response body SHOULD include `effective_budget` and `estimated_cgn`. (token-budget.md §7.4) |
| `NWP-DEPTH-EXCEEDED` | `NPS-CLIENT-BAD-PARAM` | `X-NWP-Depth` exceeds the node's permitted `max_depth` |
| `NWP-GRAPH-CYCLE` | `NPS-CLIENT-UNPROCESSABLE` | Node graph contains a cyclic reference |
| `NWP-NODE-UNAVAILABLE` | `NPS-SERVER-UNAVAILABLE` | Underlying data source is temporarily unavailable |
| `NWP-MANIFEST-VERSION-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Agent's NPS version is lower than the node's `min_agent_version` |
| `NWP-MANIFEST-NODE-TYPE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | NWM `node_type` contains the removed legacy value `"gateway"` (NPS-CR-0001); use `"anchor"` or `"bridge"`. Response SHOULD include a `hint` pointing to NPS-CR-0001. |
| `NWP-MANIFEST-NODE-TYPE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | NWM `node_type` contains an unrecognized value (not a known-removed legacy value — use `NWP-MANIFEST-NODE-TYPE-REMOVED` for that case). |
| `NWP-RATE-LIMIT-EXCEEDED` | `NPS-LIMIT-RATE` | Rate limit exceeded; reset timestamp is in the `X-NWP-Rate-Reset` header |
| `NWP-RESERVED-TYPE-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | `QueryFrame` or `SubscribeFrame` `type` field contains an unrecognized reserved-type identifier; this node does not implement the requested reserved operation (NWP §12). Distinct from `NWP-ACTION-NOT-FOUND` — use this code when the `type` field is the unknown operand, not `action_id`. |
| `NWP-TOPOLOGY-UNAUTHORIZED` | `NPS-AUTH-FORBIDDEN` | Caller lacks permission to read this Anchor's topology (NPS-2 §12); authorization policy is implementation-defined per §12.4 (NPS-CR-0002) |
| `NWP-TOPOLOGY-UNSUPPORTED-SCOPE` | `NPS-CLIENT-BAD-PARAM` | `topology.scope` value is not implemented by this Anchor Node (NPS-CR-0002) |
| `NWP-TOPOLOGY-DEPTH-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | Requested `topology.depth` exceeds this Anchor Node's configured maximum (NPS-CR-0002) |
| `NWP-TOPOLOGY-FILTER-UNSUPPORTED` | `NPS-CLIENT-BAD-PARAM` | `topology.filter` contains an unrecognized key or unsupported operator (NPS-CR-0002) |

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
| `NIP-TRUST-FRAME-INVALID` | `NPS-CLIENT-BAD-FRAME` | TrustFrame signature or format is invalid — see NPS-3 §5.2 |
| `NIP-TRUST-FRAME-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | TrustFrame `expires_at` is in the past — see NPS-3 §5.2 |
| `NIP-TRUST-FRAME-GRANTOR-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | TrustFrame `grantor_nid`'s own CA certificate is revoked or expired — see NPS-3 §5.2 |
| `NIP-TRUST-FRAME-SCOPE-EXCEEDS-GRANTOR` | `NPS-AUTH-FORBIDDEN` | TrustFrame `trust_scope` contains a capability that the grantor itself does not hold (no-scope-expansion principle) — see NPS-3 §5.2 |
| `NIP-TRUST-FRAME-NODES-PATTERN-INVALID` | `NPS-CLIENT-BAD-FRAME` | TrustFrame `nodes` entry is not a valid `nwp://` URL pattern (e.g. malformed wildcard) — see NPS-3 §5.2 |
| `NIP-REVOKE-FRAME-INVALID` | `NPS-CLIENT-BAD-FRAME` | RevokeFrame is malformed (missing required field, signature verification fails, or canonical form is invalid) — see NPS-3 §5.3 |
| `NIP-REVOKE-FRAME-UNAUTHORIZED-ISSUER` | `NPS-AUTH-FORBIDDEN` | RevokeFrame `signer_nid` is not authorised to revoke `target_nid` — see NPS-3 §5.3 |
| `NIP-REVOKE-FRAME-SERIAL-MISMATCH` | `NPS-CLIENT-BAD-PARAM` | RevokeFrame `serial` is present but does not match any currently-issued cert for `target_nid` — see NPS-3 §5.3 |
| `NIP-REVOKE-FRAME-REASON-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | RevokeFrame `reason` carries a value outside the defined enum; receivers treat it as `key_compromise` (most restrictive) — see NPS-3 §5.3 |
| `NIP-ASSURANCE-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.assurance_level` does not match the cert extension `id-nid-assurance-level` (downgrade-attack defence) — see NPS-3 §5.1.1 (NPS-RFC-0003) |
| `NIP-ASSURANCE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | `assurance_level` carries a value outside the defined enum (`anonymous` / `attested` / `verified`) — see NPS-3 §5.1.1 (NPS-RFC-0003) |
| `NIP-REPUTATION-ENTRY-INVALID` | `NPS-CLIENT-BAD-FRAME` | Reputation log entry signature fails verification or canonical (RFC 8785 JCS) form is malformed — see NPS-3 §5.1.2 (NPS-RFC-0004) |
| `NIP-REPUTATION-LOG-UNREACHABLE` | `NPS-DOWNSTREAM-UNAVAILABLE` | A log operator referenced by a Node's `reputation_policy` cannot be reached during admission evaluation — see NPS-3 §5.1.2 (NPS-RFC-0004) |
| `NIP-REPUTATION-GOSSIP-FORK` | `NPS-SERVER-INTERNAL` | Cross-peer STH consistency check failed; possible Merkle tree fork detected — see NPS-RFC-0004 §4.5 |
| `NIP-REPUTATION-GOSSIP-SIG-INVALID` | `NPS-CLIENT-BAD-FRAME` | Peer STH signature verification failed during gossip exchange — see NPS-RFC-0004 §4.5 |
| `NIP-CERT-FORMAT-INVALID` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.cert_chain` is not DER-encoded X.509 or fails ASN.1 parsing — see NPS-RFC-0002 §4.3 |
| `NIP-CERT-EKU-MISSING` | `NPS-CLIENT-BAD-FRAME` | Required NPS EKU (`agent-identity` or `node-identity`) absent or non-critical on the leaf cert — see NPS-RFC-0002 §4.1 / §4.3 |
| `NIP-CERT-SUBJECT-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | X.509 leaf cert subject CN / SAN URI does not match the `IdentFrame.nid` field — see NPS-RFC-0002 §4.3 |
| `NIP-ACME-CHALLENGE-FAILED` | `NPS-CLIENT-BAD-FRAME` | ACME `agent-01` challenge validation failed (token mismatch, signature verification failure, or replay) — see NPS-RFC-0002 §4.4 |
| `NIP-CA-GROUP-REVOKED` | `NPS-AUTH-FORBIDDEN` | Cannot issue a session under a group NID that has been revoked — see NPS-3 §5.1.3 (NPS-CR-0003) |
| `NIP-CA-PARENT-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | The `parent_nid` / group NID referenced by a session-issue request does not exist — see NPS-3 §5.1.3 (NPS-CR-0003) |
| `NIP-CA-PARENT-NOT-GROUP` | `NPS-CLIENT-BAD-PARAM` | The referenced parent NID exists but is not registered as `lineage.role = "group"` — see NPS-3 §5.1.3 (NPS-CR-0003) |
| `NIP-CA-SESSION-VALIDITY-INVALID` | `NPS-CLIENT-BAD-PARAM` | Requested session validity below 60 seconds or above the CA's configured maximum — see NPS-3 §5.1.3 (NPS-CR-0003) |
| `NIP-CA-JWS-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | Group-JWS authorisation on a session-issue request fails signature, header, or shape validation — see NPS-3 §5.1.3 (NPS-CR-0003) |
| `NIP-CA-JWS-EXPIRED` | `NPS-AUTH-UNAUTHENTICATED` | Group-JWS `iat` outside the CA's clock-skew window (default ±5 minutes) — see NPS-3 §5.1.3 (NPS-CR-0003) |
| `NIP-CERT-PARENT-REVOKED` | `NPS-AUTH-UNAUTHENTICATED` | A session NID's parent / group NID is revoked or expired (chain check, NPS-3 §7 step 3a) — see NPS-3 §5.1.3 (NPS-CR-0003) |

---

## NDP Error Codes

| Error Code | NPS Status Code | Description |
|------------|-----------------|-------------|
| `NDP-RESOLVE-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | `nwp://` address cannot be resolved to a physical endpoint |
| `NDP-RESOLVE-AMBIGUOUS` | `NPS-CLIENT-CONFLICT` | Resolution result is conflicting (multiple inconsistent registrations) |
| `NDP-RESOLVE-TIMEOUT` | `NPS-SERVER-TIMEOUT` | Resolution request timed out |
| `NDP-ANNOUNCE-SIGNATURE-INVALID` | `NPS-AUTH-UNAUTHENTICATED` | AnnounceFrame signature verification failed |
| `NDP-ANNOUNCE-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | NID in AnnounceFrame does not match the signing certificate |
| `NDP-ANNOUNCE-ROLE-REMOVED` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame `node_roles` contains the removed legacy value `"gateway"` (NPS-CR-0001); use `"anchor"` or `"bridge"`. Response SHOULD include a `hint` pointing to NPS-CR-0001. |
| `NDP-ANNOUNCE-ROLE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame `node_roles` contains an unrecognized value (not a known-removed legacy value — use `NDP-ANNOUNCE-ROLE-REMOVED` for that case). |
| `NDP-ANNOUNCE-CONFLICT` | `NPS-CLIENT-CONFLICT` | Two AnnounceFrames share the same `nid` and `graph_seq` but differ in covered content (registry-poisoning attempt; see NPS-4 §7.4) |
| `NDP-GRAPH-SEQ-ROLLBACK` | `NPS-CLIENT-BAD-FRAME` | AnnounceFrame `graph_seq` is less than or equal to the highest value previously accepted for that NID (rollback attempt; see NPS-4 §7.5) |
| `NDP-GRAPH-SEQ-GAP` | `NPS-STREAM-SEQ-GAP` | GraphFrame sequence numbers are not contiguous |
| `NDP-ISSUER-NOT-ALLOWED` | `NPS-AUTH-FORBIDDEN` | AnnounceFrame issuer (signing CA) is not in the active registry profile's issuer allowlist (see NPS-4 §7.3) |
| `NDP-CA-ATTEST-REQUIRED` | `NPS-AUTH-UNAUTHENTICATED` | Active registry profile requires a CA-attested NID and the AnnounceFrame's certificate chain does not anchor in the configured trust roots (see NPS-4 §7.3) |
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
| `NOP-RESOURCE-INSUFFICIENT` | `NPS-SERVER-UNAVAILABLE` | Preflight found one or more Worker Agents lack sufficient resources (CGN or capabilities) |
| `NOP-CONDITION-EVAL-ERROR` | `NPS-CLIENT-BAD-PARAM` | DAG node `condition` expression failed to evaluate (syntax error or missing referenced field) |
| `NOP-INPUT-MAPPING-ERROR` | `NPS-CLIENT-UNPROCESSABLE` | `input_mapping` JSONPath could not be resolved or target field is missing |
| `NOP-COMPENSATION-FAILED` | `NPS-CLIENT-UNPROCESSABLE` | Terminal — node `compensate_action` returned an error during saga rollback |
| `NOP-COMPENSATION-NOT-SUPPORTED` | `NPS-CLIENT-UNPROCESSABLE` | Terminal — predecessor that must be compensated has no `compensate_action` (and `compensation_policy="strict"`) |

---

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
