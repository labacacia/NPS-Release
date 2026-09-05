# Alpha.19 Roadmap-History Reconciliation

**Status**: Frozen for the alpha.19 debt-closure candidate

**Scope**: P19-S03, P19-D03 and P19-T03 historical statements that can be
misread as current product gaps. The original milestone facts remain in place;
this record adds their time boundary, closing milestone and current evidence.

The machine-readable record is
[`alpha19-roadmap-history-reconciliation.json`](../spec/conformance/alpha19-roadmap-history-reconciliation.json).
Run `python3 tools/scripts/check-alpha19-roadmap-history.py` to verify roadmap
markers and evidence paths.

## Claim disposition

| ID | Historical statement | True at | Current disposition |
|---|---|---|---|
| RH-01 | RFC-0001/0002/0003 used Accepted/Draft lifecycle labels | Phase 1 / alpha.5 | Preserved as a dated snapshot; current lifecycle is in the [status record](alpha19-p19-4-status-reconciliation.md). |
| RH-02 | All SDKs lacked NDP DNS TXT resolution | alpha.6 planning | Superseded by the alpha.6 six-SDK implementation and the evidence below. |
| RH-03 | RFC-0004 full-client support remained incomplete | alpha.7 planning | Superseded across alpha.13–alpha.15; all six SDKs now provide client/proof behavior, without turning every SDK into a log operator. |
| RH-04 | `nps-ingress` was only a skeleton | alpha.5/alpha.11 | Superseded by its claim-scoped TLS 1.3, ALPN, mTLS, NID-binding and proxy evidence. Full L2 certification is still not claimed. |
| RH-05 | `nps-runner` was only a skeleton and OCI/renewal remained open | alpha.5/alpha.17 | Superseded by portable OCI SpawnSpec and lease-renewal implementation. The strict L3 manifest still exposes genuine partial/unexecuted cases. |
| RH-06 | RFC-0002 was blocked on a provisional PEN/OID | alpha.5 planning | Closed on 2026-05-08 when IANA assigned PEN 65715; `1.3.6.1.4.1.99999` is historical only. |
| RH-07 | The detailed alpha.13 implementation plan described work as upcoming | alpha.11/alpha.13 planning | The file remains historical and now carries an adjacent supersession map rather than silently rewriting its plan. |

## Six-SDK DNS TXT evidence

| SDK | Source | Test |
|---|---|---|
| .NET | [`InMemoryNdpRegistry.cs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/dotnet/src/NPS.NDP/Registry/InMemoryNdpRegistry.cs) | [`NdpRegistryTests.cs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/dotnet/tests/NPS.Tests/Ndp/NdpRegistryTests.cs) |
| Python | [`registry.py`](https://github.com/labacacia/NPS-Dev/blob/main/impl/python/nps_sdk/ndp/registry.py) | [`test_ndp.py`](https://github.com/labacacia/NPS-Dev/blob/main/impl/python/tests/test_ndp.py) |
| TypeScript | [`ndp-registry.ts`](https://github.com/labacacia/NPS-Dev/blob/main/impl/typescript/src/ndp/ndp-registry.ts) | [`ndp.test.ts`](https://github.com/labacacia/NPS-Dev/blob/main/impl/typescript/tests/ndp.test.ts) |
| Go | [`registry.go`](https://github.com/labacacia/NPS-Dev/blob/main/impl/go/ndp/registry.go) | [`ndp_test.go`](https://github.com/labacacia/NPS-Dev/blob/main/impl/go/ndp/ndp_test.go) |
| Java | [`InMemoryNdpRegistry.java`](https://github.com/labacacia/NPS-Dev/blob/main/impl/java/src/main/java/com/labacacia/nps/ndp/InMemoryNdpRegistry.java) | [`NdpTest.java`](https://github.com/labacacia/NPS-Dev/blob/main/impl/java/src/test/java/com/labacacia/nps/ndp/NdpTest.java) |
| Rust | [`registry.rs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/rust/nps-ndp/src/registry.rs) | [`ndp_tests.rs`](https://github.com/labacacia/NPS-Dev/blob/main/impl/rust/nps-ndp/tests/ndp_tests.rs) |

## Daemon closure evidence

- `nps-ingress`: the [current architecture](daemons/architecture.md) and
  [`NPS-NODE-L2-TLS-EVIDENCE.json`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-ingress/conformance/NPS-NODE-L2-TLS-EVIDENCE.json)
  define the implemented transport boundary without a full-profile claim.
- `nps-runner`: the [current architecture](daemons/architecture.md),
  [`NPS-RUNNER-CAPABILITIES.json`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-runner/conformance/NPS-RUNNER-CAPABILITIES.json),
  [`SpawnSpecResolverTests.cs`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-runner/tests/SpawnSpecResolverTests.cs)
  and [`LeaseRenewalLoopTests.cs`](https://github.com/labacacia/NPS-Dev/blob/main/tools/daemons/nps-runner/tests/LeaseRenewalLoopTests.cs)
  prove the closed OCI/renewal statements while retaining real L3 limits.

## Boundary

This reconciliation changes the interpretation of historical prose, not the
facts that were true when the prose was written. It does not convert component
evidence into deployment certification, activate future protocol phases, or
publish a release.
