<!--
NPS-Node Profile L1 — Self-Attestation Template
===============================================

Copy this file to the root of the implementation-under-test (IUT) repository,
fill in every field, and sign the attestation block with the IUT's root Ed25519
private key. A passing implementation MAY then advertise L1 compliance.

See spec/services/conformance/NPS-Node-L1.md for the test suite this template
attests to.
-->

# NPS-Node Profile L1 — Certified

This implementation attests to having passed the
[NPS-Node-L1 conformance suite](https://github.com/LabAcacia/nps/blob/main/spec/services/conformance/NPS-Node-L1.md)
under the conditions recorded below.

---

## Implementation

| Field | Value |
|-------|-------|
| **Name** | _(e.g., example-daemon)_ |
| **Version** | _(semver, e.g., 0.1.0)_ |
| **Repository** | _(https URL)_ |
| **License** | _(SPDX id)_ |
| **Root NID** | `urn:nps:node:<authority>:<id>` |
| **Root public key (hex)** | _(64 hex chars, Ed25519 public key)_ |

## Peer Used for Certification

| Field | Value |
|-------|-------|
| **Name** | _(e.g., nps-dotnet-reference)_ |
| **Version** | _(e.g., 1.0.0-alpha.3)_ |
| **Source** | _(URL or package reference)_ |

## Run Environment

| Field | Value |
|-------|-------|
| **Date** | _(ISO 8601 UTC, e.g., 2026-04-24T00:00:00Z)_ |
| **Platform** | _(e.g., linux-x64, macos-arm64, win-x64)_ |
| **Hardware** | _(e.g., 1 vCPU / 1 GB RAM)_ |
| **NPS-Node Profile version** | 0.1 |
| **Conformance suite version** | 0.1 |

## Case Outcomes

_Check each box that passed; write `N/A` for optional cases the IUT declined._

### NCP — Wire format
- [ ] `TC-N1-NCP-01` — Tier-1 JSON frame round-trip
- [ ] `TC-N1-NCP-02` — Hello + Anchor handshake
- [ ] `TC-N1-NCP-03` — Loopback listener default
- [ ] `TC-N1-NCP-04` — Tier-2 negotiation hygiene

### NIP — Identity
- [ ] `TC-N1-NIP-01` — Root keypair generation and permission
- [ ] `TC-N1-NIP-02` — IdentFrame sign and verify
- [ ] `TC-N1-NIP-03` — NID format
- [ ] `TC-N1-NIP-04` — Sub-NID issuance (optional; `N/A` acceptable)

### NDP — Discovery
- [ ] `TC-N1-NDP-01` — AnnounceFrame carries activation_mode
- [ ] `TC-N1-NDP-02` — AnnounceFrame signature
- [ ] `TC-N1-NDP-03` — ResolveFrame response
- [ ] `TC-N1-NDP-04` — GraphFrame subscription (optional; `N/A` acceptable)

### NWP — Inbox and delivery
- [ ] `TC-N1-NWP-01` — Inbox accepts ActionFrame
- [ ] `TC-N1-NWP-02` — Inbox persists across restart
- [ ] `TC-N1-NWP-03` — NWP pull serves inbox
- [ ] `TC-N1-NWP-04` — 100 QPS baseline
- [ ] `TC-N1-NWP-05` — Push path (optional; `N/A` acceptable)

### Observability
- [ ] `TC-N1-OBS-01` — Frame log entry per direction
- [ ] `TC-N1-OBS-02` — Log entry fields
- [ ] `TC-N1-OBS-03` — Log destination flexibility

## Results Manifest

_Paste the JSON manifest emitted by the conformance suite here:_

```json
{
  "profile": "NPS-Node-L1",
  "profile_version": "0.1",
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

> **RFC 8785 JCS requirements**: "RFC 8785 canonicalized" means JSON Canonicalization
> Scheme (JCS). Implementations MUST use a JCS-compliant library that produces:
> (1) UTF-8 encoding with no BOM; (2) keys sorted by Unicode code point; (3) numbers
> in the specific IEEE 754 / ECMAScript serialization defined by JCS (no trailing zeros,
> no `+` sign); (4) Unicode escapes as specified in RFC 8785 §3.2.2.2. Deviations in
> any of these rules will produce a different SHA-256 and fail cross-language verification.

| Field | Value |
|-------|-------|
| **Manifest SHA-256** | _(64 hex chars)_ |
| **Signature algorithm** | `Ed25519` |
| **Signature** | _(128 hex chars, over the manifest SHA-256)_ |
| **Signed at** | _(ISO 8601 UTC)_ |

## Notes

_Optional. Describe environment-specific deviations, skipped optional cases with
rationale, or operational caveats._

---

*This attestation is self-issued. Third-party certification (NPS Cloud CA) is
targeted for L3 in 2027 Q1+ and is not available for L1 at this release.*

*Copyright: LabAcacia / INNO LOTUS PTY LTD · Apache 2.0*
