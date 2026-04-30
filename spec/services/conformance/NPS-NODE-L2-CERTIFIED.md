<!--
NPS-AaaS Profile L2 — Self-Attestation Template
================================================

Copy this file to the root of the implementation-under-test (IUT) repository,
fill in every field, and sign the attestation block with the IUT's root Ed25519
private key. A passing implementation MAY then advertise L2 compliance for the
listed scope.

See spec/services/conformance/NPS-Node-L2.md for the test suite this template
attests to. The L1 prerequisite is attested separately via NPS-NODE-L1-CERTIFIED.md.
-->

# NPS-AaaS Profile L2 — Certified

This implementation attests to having passed the
[NPS-Node-L2 conformance suite](https://github.com/LabAcacia/nps/blob/main/spec/services/conformance/NPS-Node-L2.md)
under the conditions recorded below for the scope listed.

---

## Scope of This Attestation

L2 in this release covers the topology read-back requirement only. The remaining
L2 requirements (L2-01..L2-07) are tracked in follow-up CRs.

| Requirement | CR | Covered by this attestation |
|-------------|----|----------------------------|
| L2-08 — `topology.snapshot` / `topology.stream` on Anchor Nodes | [NPS-CR-0002](https://github.com/LabAcacia/nps/blob/main/spec/cr/NPS-CR-0002-anchor-topology-queries.md) | Yes |
| L2-01..L2-07 — NOP / OTel / NPT / preflight / retry / async / AlignStream | TBD | No (future CR) |

---

## Implementation

| Field | Value |
|-------|-------|
| **Name** | _(e.g., example-anchor)_ |
| **Version** | _(semver, e.g., 0.1.0)_ |
| **Repository** | _(https URL)_ |
| **License** | _(SPDX id)_ |
| **Root NID** | `urn:nps:node:<authority>:<id>` |
| **Root public key (hex)** | _(64 hex chars, Ed25519 public key)_ |
| **L1 prerequisite** | _(URL of this implementation's NPS-NODE-L1-CERTIFIED.md, OR a date this implementation passed L1)_ |

## Peer Used for Certification

| Field | Value |
|-------|-------|
| **Name** | _(e.g., nps-dotnet-reference)_ |
| **Version** | _(e.g., 1.0.0-alpha.4)_ |
| **Source** | _(URL or package reference)_ |

## Run Environment

| Field | Value |
|-------|-------|
| **Date** | _(ISO 8601 UTC, e.g., 2026-04-27T00:00:00Z)_ |
| **Platform** | _(e.g., linux-x64, macos-arm64, win-x64)_ |
| **Hardware** | _(e.g., 1 vCPU / 1 GB RAM)_ |
| **NPS-AaaS Profile version** | 0.4 |
| **Conformance suite version** | 0.1 |

## Case Outcomes

_Check each box that passed. There are no optional cases at the L2-08 scope._

### Anchor Topology — `topology.snapshot` / `topology.stream`
- [ ] `TC-N2-AnchorTopo-01` — Snapshot of a 3-member cluster
- [ ] `TC-N2-AnchorTopo-02` — Version monotonicity across joins
- [ ] `TC-N2-AnchorTopo-03` — Sub-Anchor member surfaces with `child_anchor` and `member_count`
- [ ] `TC-N2-AnchorStream-01` — `member_joined` on NDP Announce
- [ ] `TC-N2-AnchorStream-02` — `member_left` on NDP TTL expiry
- [ ] `TC-N2-AnchorStream-03` — Resume from `topology.since_version`
- [ ] `TC-N2-AnchorStream-04` — `resync_required` when version is too old

## Results Manifest

_Paste the JSON manifest emitted by the conformance suite here:_

```json
{
  "profile": "NPS-Node-L2",
  "profile_version": "0.1",
  "scope": ["L2-08"],
  "iut": { "name": "", "version": "", "nid": "" },
  "peer": { "name": "", "version": "" },
  "run": { "date": "", "environment": "" },
  "cases": [],
  "summary": { "pass": 0, "fail": 0, "skip": 0, "na": 0 }
}
```

## Attestation

_Sign the hex-encoded SHA-256 of the results-manifest JSON (compact form, RFC 8785
canonicalized) with the IUT's root private key. Verifiers MUST check this signature
against the root public key declared above._

| Field | Value |
|-------|-------|
| **Manifest SHA-256** | _(64 hex chars)_ |
| **Signature algorithm** | `Ed25519` |
| **Signature** | _(128 hex chars, over the manifest SHA-256)_ |
| **Signed at** | _(ISO 8601 UTC)_ |

## Notes

_Optional. Describe environment-specific deviations or operational caveats._

---

*This attestation is self-issued. Third-party certification (NPS Cloud CA) is
targeted for L3 in 2027 Q1+ and is not available for L2 at this release.*

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
