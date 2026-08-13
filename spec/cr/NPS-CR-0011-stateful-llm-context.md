English | [Chinese](./NPS-CR-0011-stateful-llm-context.cn.md)

# NPS-CR-0011: Stateful LLM Context and Delta Completion

**Status**: Draft  
**Target**: v1.0.0-alpha.18  
**Date**: 2026-08-12  
**Authors**: Ori Lynn / INNO LOTUS PTY LTD  
**Tracking**: [NPS-Dev#90](https://github.com/labacacia/NPS-Dev/issues/90)  
**Touches**: NPS-2 NWP (§4.2a LLM profile, §7.5 `llm.complete`, §7.6 stateful context), NPS-3 NIP (§5.1 capabilities), unified error/status codes, NWM, six SDKs, NWP conformance vectors

---

## 1. Summary

Add an optional, provider-neutral server-side context contract to the existing
`llm.complete` action. A client can create an opaque context once, then append
only new messages and tool results while the server reuses the committed model
prefix. The contract is additive: an ordinary request with no `context` field
keeps the current stateless full-message-list semantics.

This CR does not add an NCP frame type. It adds:

1. a typed `context` request on `LlmCompleteActionRequest` for `create`,
   `append`, `fork`, and `reset`;
2. a typed context receipt on unary, async, and terminal streaming results;
3. `llm.context.status` and `llm.context.release` lifecycle actions;
4. compare-and-swap versioning, identity ownership, binding checks, atomic
   commit rules, and deterministic failures;
5. NWM discovery metadata and the NIP capability `llm:context`; and
6. usage fields that distinguish logical input, reused input, newly evaluated
   input, and serialized request bytes.

The defining safety rule is **no silent fallback**. Once a request carries a
context operation, a node MUST either execute that operation exactly or return
an ErrorFrame. It MUST NOT rebuild the full prompt and report the request as a
successful stateful/cache hit.

## 2. Motivation

A persistent NCP connection removes repeated handshakes, and MessagePack can
reduce transport bytes, but neither prevents a caller from resending the whole
conversation or a model runtime from re-evaluating it. Consequently, transport
compression alone cannot support a truthful claim about model-input token
savings.

Private provider sessions are not an interoperable answer. They differ on
ownership, stale updates, tool binding, reconnect behavior, and token accounting.
Without one NWP contract, clients either keep a provider-specific codec or infer
state from successful responses. Both choices recreate the drift that the
official `llm.complete` DTO removed.

## 3. Wire Contract

### 3.1 Existing action, additive request field

`ActionFrame.action_id` remains `llm.complete`. `ActionFrame.params` remains
`LlmCompleteActionRequest`, with one new optional field:

| Field | Type | Required | Description |
|---|---|---|---|
| `context` | `LlmContextRequestDto` | Optional | Stateful context operation. Absent means the existing stateless contract. |

When `context` is present, `ActionFrame.idempotency_key` MUST be present. The
normal 24-hour ActionFrame replay window applies and binds the key to the full
canonical request. Reusing the key for different params returns
`NWP-ACTION-IDEMPOTENCY-CONFLICT`.

If the original stateful request is still running, a duplicate receives that
conflict and MUST NOT join the live stream. After completion, unary/async replay
uses the existing cached-result rule. A completed streaming replay MUST use a
new StreamFrame sequence containing the cached logical chunks and the same
terminal context receipt; it MUST NOT run the model or mutate the context again.
Servers MAY coalesce cached text into fewer replay chunks, but the ordered text,
tool calls, stop reason, usage, and receipt MUST be logically identical.

`messages` remains required and has operation-specific meaning:

- stateless: the complete ordered conversation;
- `create` or `reset`: the complete initial replacement conversation;
- `append` or `fork`: only the ordered delta after `base_version`.

An empty `messages` array MAY be used by `fork` to clone a committed prefix.
It MUST be rejected for `create`, `append`, and `reset`.

### 3.2 `LlmContextRequestDto`

| Field | Type | Required | Description |
|---|---|---|---|
| `operation` | enum | Yes | `create`, `append`, `fork`, or `reset`. |
| `context_id` | string | Conditional | Required for `append`, `fork`, and `reset`; forbidden for `create`. Opaque, case-sensitive value. |
| `base_version` | uint64 | Conditional | Required for `append`, `fork`, and `reset`; MUST equal the current committed version. |
| `ttl_seconds` | uint32 | Optional | Requested idle lifetime. Server MAY clamp it to its advertised maximum; the receipt reports the effective expiry. |

The server generates `context_id` as an unpadded base64url string from at least
128 bits of cryptographically secure randomness. Producers emit 22–128 ASCII
characters from `[A-Za-z0-9_-]`. It MUST NOT encode a NID, tenant, model,
database key, or other security-sensitive metadata. A context ID is a locator,
never an authorization credential.

### 3.3 Context receipt

`LlmCompleteActionResponse` and the terminal `LlmCompleteStreamChunkDto` gain an
optional `context` field containing `LlmContextReceiptDto`:

| Field | Type | Required | Description |
|---|---|---|---|
| `context_id` | string | Yes | Opaque server-generated identifier. |
| `version` | uint64 | Yes | Current committed version after this completion. Starts at 1 and increases by exactly one per successful mutation. |
| `operation` | enum | Yes | Operation committed by this result: `create`, `append`, `fork`, `reset`, or `release`. A completion request accepts only the first four. |
| `state` | enum | Yes | `active` for a successful completion mutation; `released` for a successful release. |
| `expires_at` | RFC 3339 timestamp | Optional | Effective expiry, when bounded. |
| `parent_context_id` | string | Conditional | Present on `fork`; identifies the source context. |
| `parent_version` | uint64 | Conditional | Present on `fork`; identifies the immutable source snapshot. |

Stateless responses MUST omit `context`. Non-terminal stream chunks MUST omit
it. An async acknowledgment does not imply a context mutation; the receipt is
carried only by `system.task.status.result` after completion.

### 3.4 Lifecycle actions

Lifecycle operations are separate actions so a release does not masquerade as
a model completion:

| Action | Required capability | Request | Successful result |
|---|---|---|---|
| `llm.context.status` | `llm:context` | Exactly one of `context_id` or `idempotency_key` | `LlmContextStatusDto` |
| `llm.context.release` | `llm:context` | `context_id`, `base_version` | `LlmContextReceiptDto` with `state = "released"` |

`status` accepts an `idempotency_key` so a client that loses the terminal reply
to a `create` can recover the generated context ID during the ActionFrame replay
window. Status is observational and does not extend TTL. Release also requires
an ActionFrame `idempotency_key` and is idempotent for the same owner and key: a
replay returns the original release receipt.

`LlmContextStatusDto` contains `state` (`busy`, `active`, `released`, `expired`,
or `failed`), optional `context_id`, optional `version`, optional `expires_at`,
optional active `request_id`, and optional `error_code`. Active/tombstone states
MUST carry the context ID and version. An in-flight create reports `busy` but
MUST omit both until commit; a failed create reports `failed`, omits them, and
carries the terminal error code. Tombstones for released and expired contexts
SHOULD be retained for at least the node's advertised `tombstone_seconds`;
after that period the same lookup returns not-found.

## 4. State and Commit Semantics

### 4.1 State machine

```text
                  create success
  ABSENT --------------------------------> ACTIVE(v1)
                                             |
                 append/reset success        | fork success
                 ACTIVE(vN+1) <---------------+------------> ACTIVE-child(v1)
                     |                                        (parent unchanged)
                     | release / idle expiry
                     v
              RELEASED / EXPIRED tombstone ----> ABSENT
```

`append`, `reset`, and `release` are compare-and-swap operations. The supplied
`base_version` MUST equal the committed version. A node MUST serialize mutations
per context; at most one request may own the mutation reservation. A stale or
concurrent loser receives `NWP-LLM-CONTEXT-VERSION-CONFLICT` and the current
version in the ErrorFrame hint.

`fork` reads an immutable snapshot at `base_version`, creates a new context,
and leaves the parent unchanged. `reset` keeps the context ID but atomically
replaces its transcript and binding, then advances the version.

Fork atomically snapshots the parent at admission. Once admitted, a later
parent append does not invalidate that immutable snapshot. Release advances the
tombstone version from vN to vN+1; automatic expiry records the last committed
version without incrementing it.

A successful create, append, fork, or reset starts or refreshes the effective
idle TTL. Status does not refresh it. A failed or cancelled mutation does not
refresh it. A context does not expire while a valid mutation reservation is
running; if its prior TTL elapsed, an aborted reservation transitions directly
to expired after releasing the reservation.

When `ttl_seconds` is present it MUST be greater than zero. When omitted,
`create` uses the node default, `append` and `reset` preserve the context's
current effective TTL, and `fork` inherits the source context's remaining TTL.
The node MAY clamp explicit, inherited, or default values to its advertised
maximum and reports the effective bounded expiry in the receipt.

### 4.2 Atomic completion boundary

A mutation commits only when the model action reaches a successful terminal
result with a stop reason other than `error`. The committed state includes the
request delta and the terminal assistant message or tool calls. A structured
refusal represented as an ordinary `end_turn` MAY commit; a payload with
`stop_reason = "error"` does not. Validation failure, authorization failure,
provider error, timeout, cancellation, or abnormal stream termination MUST:

- leave the prior committed version and transcript unchanged;
- release any mutation reservation; and
- return the normal ErrorFrame or terminal stream error.

For streaming, the context receipt appears only in the successful terminal
chunk. Servers MUST atomically commit to the store promised by their advertised
persistence level before making that terminal receipt
observable. A disconnect can make delivery ambiguous but not the state itself:
the caller resolves ambiguity with `llm.context.status`, using the known
`context_id` or the original `idempotency_key` for a lost create response.

### 4.3 Restart and persistence

NWM advertises one persistence level:

- `connection`: context survives requests on the current NCP connection only;
- `process`: context survives reconnects routed to the same process, not a
  process restart or a request routed to another instance;
- `durable`: context survives process restart under the same logical node NID
  and endpoint identity.

A node MUST return `NWP-LLM-CONTEXT-NOT-FOUND` or
`NWP-LLM-CONTEXT-EXPIRED` after state is lost; it MUST NOT silently create a
replacement. Nodes advertising `process` or `durable` MUST use a stable
endpoint identity. Connection/process context IDs remain node-instance scoped,
and clients SHOULD pin them to the endpoint that created them. Cross-instance
shared context stores and context migration are outside this CR; a load balancer
must provide affinity unless every target is the same durable logical node.

## 5. Binding and Authorization

### 5.1 Immutable binding

At create/reset, the server binds a context to:

- the resolved model ID (after aliases/routing);
- the ordered system-message set;
- canonical tool definitions, including parameter schemas; and
- the provider/runtime compatibility revision needed to reuse the prefix.

`append` and `fork` MUST preserve that binding. They MUST reject a different
model, any new system message, or changed tool definition with
`NWP-LLM-CONTEXT-BINDING-MISMATCH`. A client that intentionally changes these
inputs uses `reset` (same context lineage) or a stateless/create request (new
lineage). Runtime-private cache handles and binding fingerprints are not sent on
the wire. On append/fork, omitted `tools` means "reuse the bound definitions";
when present, `tools` MUST canonically equal them. System-role messages are
forbidden in append/fork deltas. The required `model` field MUST resolve to the
bound model.

### 5.2 NIP ownership

Stateful `llm.complete` mutations require both `llm:complete` and
`llm:context`; streaming also requires `llm:stream`, and tools also require
`llm:tool_call` as today. Status/release require `llm:context` plus owner
authorization but not `llm:complete`, so a principal that loses model-invocation
rights can still inspect and clean up its retained state.
The context owner is the authenticated caller NID plus the node's authenticated
tenant/workspace security scope. That scope comes from the admitted identity and
deployment policy, not from a client-controlled context field.

Every lookup and mutation MUST re-run normal NIP expiry, revocation, assurance,
scope, and capability checks. A different authenticated owner receives
`NWP-LLM-CONTEXT-FORBIDDEN`. Implementations MUST perform constant-time-style
opaque-ID lookup and authorization ordering where practical; IDs remain
unguessable, but existence is not treated as a secret authorization mechanism.
Long-running generation MUST repeat revocation and authorization checks before
commit. A caller revoked or de-authorized after admission receives the normal
NIP/NWP authorization ErrorFrame and the reservation is aborted.

## 6. Discovery

`profiles.llm.profile_version` advances to `0.2`. A node supporting this CR
MUST register `llm.context.status` and `llm.context.release` in the top-level
NWM `actions` registry with `required_capability = "llm:context"`, and adds:

```json
{
  "profiles": {
    "llm": {
      "profile_version": "0.2",
      "actions": [
        "llm.complete",
        "llm.context.status",
        "llm.context.release"
      ],
      "context": {
        "supported": true,
        "operations": ["create", "append", "fork", "reset", "release"],
        "persistence": "durable",
        "max_contexts_per_principal": 32,
        "max_ttl_seconds": 3600,
        "tombstone_seconds": 86400
      }
    }
  }
}
```

Clients MUST discover `context.supported = true`, the required operation, and
the relevant `llm:context` capability before sending a stateful request. An
advertised but unavailable implementation returns
`NWP-CAPABILITY-ADVERTISED-UNIMPLEMENTED`; an operation omitted from
`operations` returns `NWP-LLM-CONTEXT-OPERATION-UNSUPPORTED`.

## 7. Usage Accounting

`LlmUsageDto` gains:

| Field | Type | Description |
|---|---|---|
| `wire_input_bytes` | uint64 | Actual complete serialized ActionFrame payload bytes accepted by the NWP decoder in the negotiated JSON/MessagePack representation, after NCP decryption and before frame-header accounting. |

The existing fields keep these meanings:

- `input_tokens`: logical model input represented by the committed prefix plus
  this request delta;
- `reused_tokens`: logical input tokens supplied from retained prefix/KV state
  without evaluation in this invocation;
- `evaluated_tokens`: input tokens newly evaluated in this invocation; and
- `output_tokens`: newly generated output tokens.

When all input counters are known,
`reused_tokens + evaluated_tokens = input_tokens`. `cache_hit = true` is valid
only when `reused_tokens > 0`. `wire_input_bytes` is measured at the NWP decoder
boundary, not re-serialized from a DTO. It includes the ActionFrame envelope
(including context and idempotency metadata) and excludes the NCP header, TLS
records, and response bytes so JSON and MessagePack results remain comparable.
These counters MUST NOT be synthesized from prompt length when the provider
cannot observe them; unavailable fields remain absent.

## 8. Errors

All failures below use ErrorFrame (or the terminal stream error), never
`LlmCompleteActionResponse.error`:

| Error | NPS status | Meaning |
|---|---|---|
| `NWP-LLM-CONTEXT-NOT-FOUND` | `NPS-CLIENT-NOT-FOUND` | Unknown context or idempotency lookup after its retention window. |
| `NWP-LLM-CONTEXT-EXPIRED` | `NPS-CLIENT-GONE` | Context has an unexpired tombstone proving idle expiry. |
| `NWP-LLM-CONTEXT-VERSION-CONFLICT` | `NPS-CLIENT-CONFLICT` | `base_version` is stale or another mutation owns the reservation. |
| `NWP-LLM-CONTEXT-BINDING-MISMATCH` | `NPS-CLIENT-CONFLICT` | Model, system prompt, tools, or runtime compatibility revision differs. |
| `NWP-LLM-CONTEXT-FORBIDDEN` | `NPS-AUTH-FORBIDDEN` | Authenticated caller is not the context owner or lacks scope/capability. |
| `NWP-LLM-CONTEXT-LIMIT-EXCEEDED` | `NPS-LIMIT-RESOURCE` | Per-principal live-context limit reached. Error hint SHOULD report the limit. |
| `NWP-LLM-CONTEXT-OPERATION-UNSUPPORTED` | `NPS-SERVER-UNSUPPORTED` | Node supports contexts but not the requested operation. |

Malformed field combinations continue to use `NWP-ACTION-PARAMS-INVALID`.
Provider/runtime failures continue to use existing NWP server errors. Released
or expired tombstones remain observable to their owner through
`llm.context.status`. Completion mutations against a released context return
not-found; mutations against an expiry tombstone return gone; after tombstone
removal both return not-found. A replay of the original release key still
follows the ActionFrame replay rule.

This CR adds `NPS-LIMIT-RESOURCE` as a generic LIMIT status (HTTP 429) for
bounded live resources such as contexts. The unified status table carries that
mapping; a live-object count is not a request-rate violation.

## 9. SDK Changes

All six SDKs implement the same wire DTOs and enum values:

- `LlmContextRequestDto`, `LlmContextReceiptDto`, `LlmContextStatusDto`;
- context operation/state enums;
- typed builders/parsers for both lifecycle actions;
- context fields on unary response and terminal stream chunks;
- `wire_input_bytes` on `LlmUsageDto`; and
- NWM `context` discovery metadata.

Server libraries additionally provide a context-store interface with atomic
reserve/commit/abort, owner validation, expiry, and status lookup. A memory
store is sufficient for `connection`/`process`; `durable` is claimed only by a
backend that passes restart tests. SDKs MUST NOT advertise
context support merely because they can encode the DTOs.

## 10. Conformance Plan

The candidate normative change adds
`spec/conformance/nwp/llm_context_vectors.json` together with NWP 0.21. The
stable case IDs are reserved below; all six SDKs must execute the file before
this CR can move from Draft to Implemented.

| ID | Required decision |
|---|---|
| `nwp.llm-context.001` | Stateless request remains byte/behavior compatible and has no receipt. |
| `nwp.llm-context.002` | Create commits v1 and returns an opaque ID only at terminal success. |
| `nwp.llm-context.003` | Append sends only delta and commits vN+1. |
| `nwp.llm-context.004` | Stale/concurrent append returns version conflict without mutation. |
| `nwp.llm-context.005` | Fork creates child v1 and leaves parent unchanged. |
| `nwp.llm-context.006` | Reset replaces binding/transcript atomically. |
| `nwp.llm-context.007` | Model/tool/system mismatch fails closed; no stateless fallback. |
| `nwp.llm-context.008` | Wrong owner or missing capability is forbidden. |
| `nwp.llm-context.009` | Cancel/timeout/stream error aborts reservation and version. |
| `nwp.llm-context.010` | Lost create response is resolved by idempotency-key status lookup. |
| `nwp.llm-context.011` | Release/expiry and tombstone error transitions are deterministic. |
| `nwp.llm-context.012` | Usage equation and measured wire bytes are internally consistent. |
| `nwp.llm-context.013` | NWM claims exactly the implemented operations/persistence level. |
| `nwp.llm-context.014` | Restart behavior matches `connection`/`process`/`durable`. |
| `nwp.llm-context.015` | Idempotent streaming replay does not regenerate or recommit. |
| `nwp.llm-context.016` | Revocation before commit aborts the reservation and mutation. |
| `nwp.llm-context.017` | Per-principal context limit uses `NPS-LIMIT-RESOURCE` and does not allocate. |
| `nwp.llm-context.018` | An unadvertised operation returns operation-unsupported. |
| `nwp.llm-context.019` | Missing stateful idempotency key is params-invalid before dispatch. |

An end-to-end benchmark MUST compare the same multi-turn conversation in
stateless and stateful modes. On the second turn, stateful mode MUST send only
the delta, report lower `wire_input_bytes`, and report lower
`evaluated_tokens`, while preserving ordered role/tool semantics. The benchmark
MUST use strict native mode with protocol fallback disabled.

## 11. Migration and Compatibility

- Existing clients, servers, manifests, and `llm.complete` payloads remain valid.
- A client must opt in by sending `context`; support is never inferred from a
  persistent connection or `cache_hit`.
- A context-capable client can fall back to stateless only before submitting a
  context operation and only by explicit local policy. A server cannot perform
  that fallback on the client's behalf.
- Context IDs are provider-neutral at the NWP boundary but not portable between
  unrelated logical node identities.

## 12. Out of Scope

- Standardizing a model's internal KV-cache layout or moving cache tensors
  between providers.
- Defining shared long-term memory, retrieval, or NDP node discovery semantics.
- Exposing hidden chain-of-thought or provider-private reasoning state.
- Guaranteeing bit-identical model output across stateless and stateful runs.
- Cross-owner context sharing or delegated context transfer. A later CR may add
  an explicit grant object; bearer-style context sharing is forbidden here.
- Cross-instance context migration or a distributed context-store protocol.
- Publishing the NPS LLM Framework or Willow as part of alpha.18.

## 13. Acceptance Criteria

- [ ] Independent design review accepts operation names, commit boundary,
      ownership, persistence levels, and errors.
- [ ] NWP/NIP/error/status specs land in English and Chinese.
- [ ] All six SDKs expose matching DTOs and NWM metadata.
- [ ] All six SDKs run the shared context vectors in CI.
- [ ] At least one native server passes cancel, reconnect, restart, and
      concurrent-update integration tests.
- [ ] Strict-native Ivy integration deletes its private stateful payload codec.
- [ ] The stateless path has no behavioral regression.
- [ ] The benchmark demonstrates both transport-byte and evaluated-token savings.
- [ ] Source-of-truth and version-consistency gates are green before distribution.

## 14. Proposed CHANGELOG Entry

> **NWP stateful LLM context (NPS-CR-0011, Draft)**: add an opt-in context/delta
> contract for `llm.complete`, opaque owner-bound context IDs, CAS versions,
> create/append/fork/reset/status/release lifecycle, atomic stream completion,
> NWM/NIP discovery and authorization, deterministic errors, and measured
> logical/reused/evaluated/wire usage. Stateless completion remains unchanged;
> stateful requests never silently fall back.

## 15. Resolved Design Choice

Lost-response recovery uses the existing mandatory 24-hour ActionFrame
idempotency window. This CR does not introduce a second retention knob: a node
must retain the owner-scoped key-to-outcome record for that full window even if
the associated context has a shorter TTL. The record may point to an expired
or released tombstone; after 24 hours, key lookup may return not-found.
