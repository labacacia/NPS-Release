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
├── NODE-PROFILE-CASE-INVENTORY.md       # Node L1/L2 evidence map (not certification)
├── NODE-PROFILE-CASE-INVENTORY.cn.md    # Chinese evidence-map companion
├── node-profile-case-inventory.json     # machine-readable case/evidence inventory
├── node-profile-implementation-manifests.json # advertised implementation manifest registry
├── aaas-l2-requirement-disposition.json # current-contract L2-01..L2-07 strength/case/evidence map
├── alpha19-status-reconciliation.json    # CR/RFC lifecycle, SDK coverage and question disposition
├── alpha19-roadmap-history-reconciliation.json # dated roadmap claims, closures and current evidence
├── alpha19-six-sdk-validation.json       # P19-5 six-language test-matrix evidence
├── alpha19-resilience-validation.json    # P19-5 NativeAOT, race, restart and partition evidence
├── ncp/
│   ├── anchor_id_vectors.json           # AnchorFrame anchor_id (RFC 8785 JCS + SHA-256)
│   ├── binary_vector_payload_vectors.json # Tier-3 BinaryVector v1 payload layout + malformed-payload cases
│   ├── encoding_policy_vectors.json     # Negotiated encoding policy + unnegotiated Tier-3 rejection
│   ├── frame_header_vectors.json        # 4-byte / 8-byte fixed header encode + decode
│   ├── hello_caps_vectors.json          # HelloFrame ↔ CapsFrame capability negotiation
│   ├── native_server_handshake_vectors.json # Native admission, bounds, and deterministic negotiation
│   └── runtime_hardening_vectors.json    # Alpha.19 keepalive, close, QUIC, and backpressure decisions
├── nip/
│   ├── ident_signature_vectors.json     # IdentFrame Ed25519 signature canonical form
│   ├── revocation_policy_vectors.json   # Ordered live-revocation and fail-open/closed policy
│   ├── scope_matching_vectors.json      # NID scope.nodes glob match (positive + negative)
│   ├── signed_crl_vectors.json           # Deterministic signed CA revocation artifacts
│   └── renewal_revocation_vectors.json  # Alpha.19 renewal, freshness, OCSP, and advisory decisions
├── nwp/
│   ├── filter_dsl_vectors.json          # QueryFrame filter DSL parse + evaluate
│   ├── action_frame_vectors.json        # ActionFrame: idempotency, async lifecycle, system.task.*, callback_url
│   ├── llm_context_vectors.json         # Stateful LLM context: CAS, ownership, lifecycle, replay, usage
│   ├── subscribe_frame_vectors.json     # SubscribeFrame: seq monotonicity, cursor, SSE wire, topology.stream events
│   ├── query_frame_aggregation_vectors.json  # QueryFrame aggregation + topology.snapshot shape
│   ├── alpha19_hardening_vectors.json   # Alpha.19 NWM normalization + renewable subscription leases
│   ├── portable_node_server_vectors.json # HTTP/native Node admission and dispatch profile
│   └── bridge_lifecycle_vectors.json     # Bridge preflight, SSRF, deadline, cancellation, correlation
├── ndp/
│   ├── announce_canonicalization_vectors.json # Announce signed body + Ed25519 verification
│   ├── registry_consistency_vectors.json      # Registry convergence, expiry, epoch, and Bridge discovery
│   └── recovery_fence_vectors.json       # Alpha.19 durable restart/partition fence decisions
└── nop/
    ├── dag_validation_vectors.json      # DAG cycle detection + max-node + chain-depth
    ├── orchestrator_transcripts.json    # Deterministic DAG/retry/aggregation/saga sessions
    ├── runtime_security_vectors.json    # Callback, lease, delegation, SpawnSpec, lifecycle
    └── replay_retention_vectors.json    # Alpha.19 replay ledger, TTL, eviction, and aggregation
```

`ndp/` carries the NDP 0.12 Registry Conformance profile. Its transcript
vectors begin with an empty registry and assert deterministic admission
decisions, replay fences, live contents, cluster resolution, and Bridge
capability discovery.

`nop/` carries the NOP 0.9 Orchestrator Conformance profile. Its transcripts
assert deterministic event order and terminal results, while the runtime
vectors cover fail-closed callback validation, HMAC, Anchor re-resolution,
scope carving, leases, SpawnSpec validation, lifecycle limits, and dedup keys.

`nwp/llm_context_vectors.json` carries the NWP 0.21 stateful LLM context
profile. It asserts stateless compatibility, owner-bound opaque IDs, atomic
CAS/commit/abort, lifecycle tombstones, restart truth, idempotent stream replay,
authorization re-checks, and measured token/wire accounting.

The five `alpha19`/hardening files freeze the P19-1 design boundary before any
six-SDK port begins. Together they cover 46 stable requirement IDs from NCP
0.12, NWP 0.22, NIP 0.15, NDP 0.13, and NOP 0.10. The traceability matrix is
[`docs/alpha19-p19-1-traceability.md`](../../docs/alpha19-p19-1-traceability.md).

The root-level Node profile inventory maps every Node L1/L2 case heading to
current reference-repository evidence and an explicit gap status. It is an
evidence index, not a protocol vector set and not a certification claim; the
normative service-tier profiles remain under `spec/services/conformance/`.
`node-profile-implementation-manifests.json` separately enumerates every
repository implementation that advertises profile or family evidence, the
manifest that records its complete claimed scope, and whether certification is
explicitly withheld.
`aaas-l2-requirement-disposition.json` separately freezes each pre-alpha.20
AaaS L2-01..L2-07 requirement, its normative strength, stable v0.7 case ID,
reference component evidence and full-claim rule. Run
`tools/scripts/check-aaas-l2-disposition.py` to verify the two language specs,
self-attestation templates, evidence paths and all six SDK catalogs together.
`alpha19-status-reconciliation.json` records current CR/RFC lifecycle,
six-SDK evidence, resolved-question disposition and deliberately inactive
future phases. Run `tools/scripts/check-alpha19-status-reconciliation.py` to
verify indexes, bilingual RFC bodies, coverage tables and evidence paths.
`alpha19-roadmap-history-reconciliation.json` records when superseded roadmap
claims were true, what closed them and which current source/tests replace them.
Run `tools/scripts/check-alpha19-roadmap-history.py` to verify the historical
markers, six-SDK DNS evidence and closed PEN/daemon statements.
`alpha19-six-sdk-validation.json` records the P19-5 full language-test matrix,
exact commands, counts, exclusions and security observations. Run
`tools/scripts/check-alpha19-six-sdk-validation.py` to validate the evidence
shape and totals.
`alpha19-resilience-validation.json` records the P19-5 NativeAOT and Go race
results, repeated runner/npsd resilience cases, six-language NDP partition-fence
matrix and explicit non-certification boundary. Run
`tools/scripts/check-alpha19-resilience-validation.py` to validate the evidence
shape and totals.

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
      "requirement": "<normative requirement ID or comma-separated IDs>",
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
2. For every protocol-vector `.json` file under the `ncp/`, `nip/`, `nwp/`,
   `ndp/`, and `nop/` subdirectories, load all vectors and run the corresponding
   code path (encode/decode/sign/match/validate). The root-level
   `node-profile-case-inventory.json` is metadata and is not a vector file.
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

## Bilingual documentation gate

The alpha.19 EN/CN contract is recorded in
[`alpha19-bilingual-parity.json`](./alpha19-bilingual-parity.json). Run
`tools/scripts/check-alpha19-bilingual-parity.py --release-root ../NPS-Release`
from the repository root to validate every bilingual Markdown pair in both the
source candidate and the published release snapshot.

## Security validation gate

The alpha.19 point-in-time dependency, workflow, and Git-history security
record is [`alpha19-security-validation.json`](./alpha19-security-validation.json).
Run `tools/scripts/check-alpha19-security-validation.py` to validate the
recorded ecosystem results, residual-risk disclosure, and immutable action
references. The live scanners are defined in `.github/workflows/security.yml`.

## Package dry-run gate

The reversible alpha.19 package-family record is
[`alpha19-package-dry-run.json`](./alpha19-package-dry-run.json). Run
`tools/scripts/check-alpha19-package-dry-run.py` to validate the NuGet, npm,
PyPI, Maven, Go and Rust inventories, migrated workspace-path mappings, and
the explicit `ready to sync` rather than `ready to tag` boundary.

---

**Maintainer**: spec WG
**Issues**: [`spec`, `testing`, `interop`] labels on `labacacia/NPS-Dev`
