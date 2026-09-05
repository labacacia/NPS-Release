# Alpha.19 CR/RFC Status Reconciliation

**Status**: Frozen for the alpha.19 debt-closure candidate

**Scope**: Current lifecycle, SDK coverage and question disposition; historical
roadmap correction is a separate P19-4 task.

The machine-readable record is
[`alpha19-status-reconciliation.json`](../spec/conformance/alpha19-status-reconciliation.json).
Run `python3 tools/scripts/check-alpha19-status-reconciliation.py` to verify it
against the specifications, indexes and source evidence.

## Lifecycle truth

| Record | Current status | Reason |
|---|---|---|
| NPS-CR-0011 | Implemented | NWP 0.21, 19 shared vectors, six SDKs and server/benchmark evidence are present; both CR indexes now match the body. |
| NPS-RFC-0001 | Active | The .NET reference helper landed in Phase 1 and all six SDK preamble helpers/tests are present. Phase 3/4 compatibility transitions are not active. |
| NPS-RFC-0002 | Active | X.509 and ACME `agent-01` source/tests exist across six SDKs and six CA surfaces. The v2 default flip and v1 removal are not active. |
| NPS-RFC-0003 | Active | Six SDKs implement Phase 1–2 assurance types, issuance, verification and opt-in enforcement. The Phase 3 flag day is excluded from alpha.19. |
| NPS-RFC-0004 | Active | All six SDKs have client/proof behavior; only .NET plus `nps-ledger` claims the reference operator and gossip implementation. |
| NPS-RFC-0005 | Active | The accepted availability, process-local ban and fail-most-restrictive defaults remain current. |
| NPS-RFC-0006 | Accepted | English and Chinese statuses now agree. Candidate daemon evidence does not silently promote the RFC to Active. |

## SDK-matrix truth

RFC-0001 through RFC-0004 no longer contain anonymous `_TBD_` owners or stale
`pending` cells. Every implemented cell names a maintained source/test surface.
The tables are capability-scoped: an RFC-0004 client/proof row is not a log-
operator claim.

CR-0003 and CR-0005 now record their six implemented CA surfaces. CR-0006
records `SubscribeFrame` and cursor support in all six SDKs while retaining the
real non-normative gap: Python and Rust do not expose a high-level subscribe
client convenience API.

## Question disposition

Proposal-era defaults are now decisions or explicit future scope. Notable
current-contract choices are:

- RFC-0001 keeps the eight-byte client-to-server preamble and fixed 10-second
  read timeout; RFC-0006 owns ALPN `nps/1.0`.
- RFC-0003 does not standardize legal-entity X.509 fields in Phase 1–2, and an
  executing Node independently enforces `min_assurance_level`.
- RFC-0004 stores signed entries plus evidence hashes/references, not blobs;
  its append-only protocol has no TTL.
- RFC-0005 keeps `on_log_unavailable=allow`, process-local portable ban state
  and the most-restrictive multi-log rule.
- RFC-0006 binds the current QUIC contract to QUIC v1; QUIC v2 needs a future
  RFC amendment.

These decisions do not activate the explicitly deferred phase transitions in
the machine-readable record and do not claim release publication.
