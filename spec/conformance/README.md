# NPS Cross-Language Conformance Vectors

**Status**: Draft v0.1
**Date**: 2026-05-09
**Tracking**: [NPS-Dev#38](https://github.com/labacacia/NPS-Dev/issues/38)

---

## Purpose

This directory contains language-agnostic JSON test vectors that every NPS SDK
(.NET / Python / TypeScript / Java / Rust / Go) MUST run in CI to verify
cross-language wire-format interoperability.

The vectors are derived directly from the protocol specs (`spec/NPS-*.md`) and
are the **single source of truth** for protocol-level conformance. SDK-internal
test suites cover language-specific concerns (API ergonomics, error types,
async behaviour); these shared vectors cover the bytes on the wire and the
deterministic computations every implementation MUST agree on.

Two SDKs that both pass these vectors are guaranteed to produce frames the
other can decode for the cases listed.

## Layout

```
spec/conformance/
├── README.md                            # this file — vector format + consumption rules
├── ncp/
│   ├── anchor_id_vectors.json           # AnchorFrame anchor_id (RFC 8785 JCS + SHA-256)
│   ├── frame_header_vectors.json        # 4-byte / 8-byte fixed header encode + decode
│   └── hello_caps_vectors.json          # HelloFrame ↔ CapsFrame capability negotiation
├── nip/
│   ├── ident_signature_vectors.json     # IdentFrame Ed25519 signature canonical form
│   └── scope_matching_vectors.json      # NID scope.nodes glob match (positive + negative)
├── nwp/
│   ├── filter_dsl_vectors.json          # QueryFrame filter DSL parse + evaluate
│   ├── action_frame_vectors.json        # ActionFrame: idempotency, async lifecycle, system.task.*, callback_url
│   ├── subscribe_frame_vectors.json     # SubscribeFrame: seq monotonicity, cursor, SSE wire, topology.stream events
│   └── query_frame_aggregation_vectors.json  # QueryFrame aggregation + topology.snapshot shape
├── ndp/
│   └── (reserved — Phase 2)
└── nop/
    └── dag_validation_vectors.json      # DAG cycle detection + max-node + chain-depth
```

`ndp/` is reserved for Phase 2 (Announce / Query frame vectors land with the
NDP federation work; see `spec/NPS-Roadmap.md`).

## Vector file format

Every vector file is a single top-level JSON object with this shape:

```json
{
  "name":        "<short identifier, lowercase + dashes>",
  "version":     "<vector-set semver, e.g. \"0.1.0\">",
  "spec_ref":    "<path + section, e.g. \"spec/NPS-1-NCP.md §4.1\">",
  "description": "<one-paragraph human summary>",
  "vectors": [
    {
      "id":          "<unique-within-file, e.g. \"ncp.anchor_id.001\">",
      "description": "<what this case proves>",
      "kind":        "positive | negative",
      "input":       { ... },
      "expected":    { ... }
    }
  ]
}
```

Conventions:

- `id` is stable across vector revisions — additions append, never renumber.
- `kind: "positive"` means the SDK MUST produce the `expected` output (or the
  `expected.match=true` decision); `kind: "negative"` means the SDK MUST reject
  the input with the `expected.error` code.
- `input` and `expected` shapes are vector-set-specific; each file's
  description explains the schema.
- All bytes are encoded as **lowercase hex strings**, no `0x` prefix, no
  separators.
- All canonical JSON outputs are encoded as UTF-8 strings exactly as the SDK
  must produce them — no leading/trailing whitespace.
- Time values are RFC 3339 / ISO 8601 UTC with `Z` suffix.

## How an SDK consumes the vectors

Each SDK CI workflow MUST:

1. Check out this directory (vendored, submodule, or fetched at CI start).
2. For every `.json` file under `spec/conformance/`, load all vectors and run
   the corresponding code path (encode/decode/sign/match/validate).
3. For each `kind: "positive"` vector — assert SDK output bit-equals
   `expected`.
4. For each `kind: "negative"` vector — assert the SDK rejects the input and
   emits the protocol error code in `expected.error`.
5. Fail the build if any vector fails. The CI step name SHOULD be
   `conformance-vectors` so dashboards can group across SDKs.

A skeleton runner per language lives in each SDK's `tests/conformance/`
directory; the runner is intentionally thin (load JSON → drive SDK → diff)
because the vectors carry the protocol semantics.

### Versioning

- The vector set carries an independent semver (`version` field per file).
- **Patch** (0.1.0 → 0.1.1): add new vectors, fix typos in `description`.
  SDKs MUST keep passing.
- **Minor** (0.1.0 → 0.2.0): add new optional fields to vector schemas,
  add new vector files. SDKs MAY need updates.
- **Major** (0.1.0 → 1.0.0): change vector schema or remove vectors. SDKs
  MUST update.

The version of a vector set follows the underlying spec section's version
where possible — e.g. `ncp/anchor_id_vectors.json` versioning tracks
`NPS-1-NCP.md §4.1`.

## Adding a new vector

1. Pick the right file (or create a new one matching the layout above).
2. Choose a stable `id` continuing the file's numbering.
3. Provide both `input` and `expected`. Compute `expected` by running a
   reference implementation **and** by hand-deriving the result from the spec
   — they must agree.
4. Add a `description` that names the spec rule the case proves.
5. Bump the `version` field per the rules above.
6. Open a PR with the spec change and vector change in the same commit when
   possible.

## Negative-case discipline

Negative vectors are as important as positive ones — the `interop` failure
mode is "SDK A emits a malformed frame, SDK B accepts it anyway." Every
positive vector SHOULD have at least one negative-case sibling that mutates
one byte / one field / one bit and asserts rejection with a specific protocol
error code (`NCP-*`, `NIP-*`, `NWP-*`, `NOP-*` per `spec/error-codes.md`).

## Out of scope

These vectors do **not** cover:

- Service-tier conformance (Anchor Node L1/L2/L3 admission, AaaS billing
  flows). Those live under `spec/services/conformance/`.
- Performance benchmarks. Those live under `docs/benchmarks/`.
- TLS / transport behaviour. Tested per-SDK against real TLS stacks.
- Free-form fuzz corpora. SDKs SHOULD maintain their own.

---

**Maintainer**: spec WG
**Issues**: [`spec`, `testing`, `interop`] labels on `labacacia/NPS-Dev`
