English | [中文版](./NODE-PROFILE-CASE-INVENTORY.cn.md)

# Node L1/L2 Case Evidence Inventory

**Status**: Alpha.19 debt-closure evidence map
**Date**: 2026-09-05
**Machine-readable source**: [`node-profile-case-inventory.json`](./node-profile-case-inventory.json)

This inventory distinguishes a case title in a catalog from executable
conformance evidence. It does not certify an implementation. Certification
still requires the suite's complete IUT, peer, environment, result manifest,
and attestation rules.

## Classification

| Class | Meaning |
|---|---|
| `executable_iut` | A runnable test exercises the case against the named implementation boundary. |
| `component_executable` | Runnable component evidence exists, but the complete paired-peer IUT fixture does not. |
| `partial` | Some acceptance criteria execute; at least one criterion or fixture is missing. |
| `catalog_only` | The case is specified and catalogued, with no case-complete executable evidence. |
| `not_applicable_reference_iut` | The optional capability is explicitly declined by the reference IUT only. |

`component_executable` and `partial` are not passes. A catalog entry is metadata,
not a test result.

## Result

| Profile | Spec headings | .NET catalog | IUT executable | Component only | Partial | Catalog only | Reference-IUT N/A |
|---|---:|---:|---:|---:|---:|---:|---:|
| Node L1 v0.1 | 20 | 20 | 8 | 0 | 6 | 5 | 1 |
| Node L2 v0.7 | 38 | 38 | 6 | 16 | 11 | 5 | 0 |

The L1 suite contains 20 case headings, not 21. The old prose count was an
editorial error and is corrected without changing the case set. The old .NET
L2 catalog contained only 16 topology/TLS cases; v0.6 closed that drift at 31.
V0.7 now contains all 38 headings after adding the current-contract
`TC-N2-AaaS-01..07` cases for AaaS L2-01..L2-07.

## Immediate executable gaps

- L1: general frame echo, literal default-port isolation, complete root identity
  and peer-Ident cases, ResolveFrame unknown-target behavior, optional Graph,
  the specified 100-QPS environment, and structured per-direction frame logs.
- L2 topology: depth-cap rejection, real NDP-driven join/TTL leave, snapshot
  filter rejection, and an external paired-peer Anchor fixture.
- L2 Bridge: conformance-ID harnesses, ambiguity rejection, complete foreign
  error mapping, and undeclared direction/protocol refusal.
- L2 HA: terminal failover, quorum-driven read-only behavior, standby write
  rejection, stale-leader fencing, and the complete single-Anchor no-event case.

The exact per-case classification, evidence paths, and gaps are in the JSON
inventory and are mechanically checked against both Markdown suites and the
.NET catalog.

## Non-claims

- No full Node L1 or L2 certification is inferred from this inventory.
- Component tests are not relabeled as paired-peer IUT passes.
- An N/A disposition for npsd's optional L1 push path does not exempt another IUT.
- No version, tag, package, image, or release publication is implied.
