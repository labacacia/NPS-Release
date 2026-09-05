# NPS alpha.19 P19-1 specification and fixture freeze

**Status**: Review candidate
**Date**: 2026-08-31
**Parent inventory**: [P19-0 debt inventory](alpha19-debt-inventory.md)
**Governance**: [ChangeControl EPIC-004](https://github.com/innolotus/ChangeControl/issues/4)

## Purpose

P19-1 turns the accepted alpha.19 debt boundary into testable protocol
requirements before runtime work begins. It freezes 46 stable requirement IDs
and five shared semantic vector sets. A vector file is a contract for the
future six-SDK runners; its presence does not claim that a runner executes it.

## Traceability matrix

| Protocol | Candidate version | Debt IDs | Normative requirements | Shared vectors | Product issue |
|---|---:|---|---|---|---|
| NCP | 0.12 | P19-P01, P19-P06 | NCP-P19-01–NCP-P19-08 | `spec/conformance/ncp/runtime_hardening_vectors.json` | [#92](https://github.com/labacacia/NPS-Dev/issues/92) |
| NWP | 0.22 | P19-P04, P19-S01 | NWP-P19-01–NWP-P19-10 | `spec/conformance/nwp/alpha19_hardening_vectors.json` | [#95](https://github.com/labacacia/NPS-Dev/issues/95) |
| NIP | 0.15 | P19-P05, P19-T01, P19-S01 | NIP-P19-01–NIP-P19-10 | `spec/conformance/nip/renewal_revocation_vectors.json` | [#95](https://github.com/labacacia/NPS-Dev/issues/95) |
| NDP | 0.13 | P19-P02, P19-S01, P19-S03 | NDP-P19-01–NDP-P19-08 | `spec/conformance/ndp/recovery_fence_vectors.json` | [#93](https://github.com/labacacia/NPS-Dev/issues/93) |
| NOP | 0.10 | P19-P03, P19-S01 | NOP-P19-01–NOP-P19-10 | `spec/conformance/nop/replay_retention_vectors.json` | [#94](https://github.com/labacacia/NPS-Dev/issues/94) |

## Compatibility decisions

- NCP 0.12 changes runtime policy, not frame layout. QUIC/TLS 0-RTT NPS data is
  rejected because it is replay-sensitive; confirmed-handshake retry remains.
- NWP 0.22 adds optional manifest and SubscribeFrame fields. A v0.21 producer
  that omits `subscription_policy` retains unleased compatibility behavior.
- NIP 0.15 adds no frame field. It freezes CA/verifier timing and result policy,
  and keeps the Phase-3 flag day disabled.
- NDP 0.13 adds no frame field. Durable recovery is mandatory only for the two
  persistence-required Registry profiles; `local-dev` remains explicitly
  volatile.
- NOP 0.10 adds server-side replay policy and errors without changing existing
  TaskFrame layout. Existing result TTL remains the result-retention input.
- Minimum-compatible floors remain unchanged until executable six-SDK evidence
  shows that raising a floor is necessary.

## P19-1 review gate

P19-1 is reviewable when all of the following are true:

- every requirement ID exists once in EN and once in CN with equivalent force;
- every requirement ID is referenced by at least one valid JSON vector;
- the version matrix, EN/CN headers, dependencies, README badges/tables,
  `frame-registry.yaml`, error namespace, and maintainer table agree;
- positive, negative, boundary, restart/partition, and failure cases are
  represented where applicable;
- no spec text claims that the six SDKs already execute the new vectors;
- no tag, package, image, compatibility floor, or alpha.20-only design changes.

P19-2 starts only after this review unit is accepted. Runtime implementation
and per-language vector adapters belong to P19-2, not this document.
