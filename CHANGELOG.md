English | [中文版](./CHANGELOG.cn.md)

# Changelog — NPS-Release

This repository archives the canonical spec documents and GitHub Pages site for each NPS suite release.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [Unreleased]

### Fixed

- **`spec/error-codes.md` v0.9 → v1.0**: Four NWP topology error codes (`NWP-TOPOLOGY-UNAUTHORIZED`, `NWP-TOPOLOGY-UNSUPPORTED-SCOPE`, `NWP-TOPOLOGY-DEPTH-UNSUPPORTED`, `NWP-TOPOLOGY-FILTER-UNSUPPORTED`) were defined in NPS-2 NWP §13 and referenced by NPS-CR-0002 §3.5 but omitted from the error-codes registry during the alpha.4 spec sync. Registrations added; wording is consistent with NWP §13 definitions.

---

## [1.0.0-alpha.4] — 2026-04-30

### Spec

- **NPS-CR-0002 — Implemented (pre-1.0 fast-track)**: Reserved query types
  `topology.snapshot` and `topology.stream` on Anchor Nodes, mandatory at
  NPS-AaaS-Profile L2. New top-level NWP §12 covers both query types;
  QueryFrame §6.1 and SubscribeFrame §8.1 each gain an optional `type` field;
  DiffFrame §8.2 `event_type` enum extends to topology events. Four new error
  codes added to `spec/error-codes.md`. New §14.7 Topology Read-back security
  section. AaaS-Profile §4.3 gains L2-08 mandating both query types on Anchor
  Nodes that maintain a member registry. New conformance suite
  `spec/services/conformance/NPS-Node-L2.md` v0.1 with seven
  `TC-N2-AnchorTopo-*` / `TC-N2-AnchorStream-*` test cases; companion
  `NPS-NODE-L2-CERTIFIED.md` self-attestation template.

- **NPS-RFC-0002 — Prototype landed (status: Draft)**: `IdentFrame` gains
  optional `cert_format` (`"v1-proprietary"` default | `"v2-x509"`) and
  `cert_chain` (base64url DER) fields — non-breaking dual-trust extension; v1
  verifiers ignore the new fields and remain valid. Four new NIP error codes
  (`NIP-CERT-FORMAT-INVALID`, `NIP-CERT-EKU-MISSING`,
  `NIP-CERT-SUBJECT-NID-MISMATCH`, `NIP-ACME-CHALLENGE-FAILED`) added to
  `spec/error-codes.md`. RFC remains Draft pending shepherd review; promotion
  to Proposed/Accepted blocked on cross-SDK port wave completion + IANA PEN
  (`nid-assurance-level` currently under provisional OID `1.3.6.1.4.1.99999`).

### Version bumps (vs alpha.3)

| Doc | alpha.3 | alpha.4 |
|-----|---------|---------|
| NPS-1 NCP | v0.6 | v0.6 (unchanged) |
| NPS-2 NWP | v0.7 | v0.8 |
| NPS-3 NIP | v0.5 | v0.5 (unchanged) |
| NPS-4 NDP | v0.5 | v0.5 (unchanged) |
| NPS-5 NOP | v0.4 | v0.4 (unchanged) |
| NPS-AaaS-Profile | v0.3 | v0.4 |
| NPS-Node-Profile | v0.1 | v0.1 (unchanged) |
| NPS-Node-L2 conformance | n/a | v0.1 (new) |
| frame-registry | v0.9 | v0.9 (unchanged) |
| error-codes | v0.8 | v0.9 |
| status-codes | v0.4 | v0.4 (unchanged) |
| token-budget | v0.2 | v0.2 (unchanged) |

### GitHub Pages

- README + Pages site sources updated to `v1.0.0-alpha.4` throughout.

---

## [1.0.0-alpha.3] — 2026-04-26

### Spec — major content release

This is the first **content-rich** alpha-cycle release: spec documents
moved from "Proposed with placeholder companion artifacts" at alpha.2
to "Proposed + four accepted Change-style artifacts and a substantially
expanded surface".

- **Layout flattened**: `spec/protocols/` directory removed; the
  `NPS-{0..5}-*.md` files now live directly under `spec/` to match
  the canonical layout in the development monorepo `LabAcacia/nps`.
  External links of the form `spec/protocols/NPS-1-NCP.md` should be
  updated to `spec/NPS-1-NCP.md`.

- **NPS-CR-0001 — Implemented**: legacy `Gateway Node` (NWP) split
  into **Anchor Node** (cluster control plane + NOP routing —
  renamed inheritance) and **Bridge Node** (NPS↔non-NPS protocol
  translation — new). The wire value `node_type: "gateway"` is
  removed; parsers MUST reject it. NWP §2.1 rewritten with full
  Anchor + Bridge subsections; AaaS-Profile §2 renamed Gateway →
  Anchor and gained §2A Bridge Node. NDP `Announce` gained
  `node_kind` / `cluster_anchor` / `bridge_protocols`. The
  inverted-direction adapters historically published as
  `NPS-{mcp,a2a,grpc}-bridge` (now `NPS-*-ingress`) are documented
  in `spec/cr/NPS-CR-0001-anchor-bridge-split.md`.

- **NPS-RFC-0001 — Accepted (Phase 1)**: NCP native-mode 8-byte
  connection preamble `b"NPS/1.0\n"`. New §2.6.1 in `NPS-1-NCP.md`.
  Adds `NCP-PREAMBLE-INVALID` error code + new `PROTO` status
  category with `NPS-PROTO-PREAMBLE-INVALID`. Frame-type byte
  `0x4E` reserved.

- **NPS-RFC-0003 — Accepted (Phase 1)**: three-tier Agent identity
  assurance levels (`anonymous` / `attested` / `verified`) for
  anti-scraping and trust gating. New §5.1.1 in `NPS-3-NIP.md`;
  `IdentFrame` gains optional `assurance_level`; NWM gains optional
  `min_assurance_level` (top-level + per-action override). New
  error codes `NIP-ASSURANCE-MISMATCH` / `NIP-ASSURANCE-UNKNOWN` /
  `NWP-AUTH-ASSURANCE-TOO-LOW`. The X.509 critical-extension flip
  is gated behind RFC-0002 in alpha.4.

- **NPS-RFC-0004 — Accepted (Phase 1)**: append-only NID reputation
  log (Certificate Transparency for Agents). New §5.1.2 in
  `NPS-3-NIP.md` defines the 12-field signed entry shape, the 8-value
  `incident` enum, the 5-step `severity` ladder, and the RFC 8785
  (JCS) dual-signature rule. New error codes
  `NIP-REPUTATION-ENTRY-INVALID` / `NIP-REPUTATION-LOG-UNREACHABLE`,
  new status code `NPS-DOWNSTREAM-UNAVAILABLE`. Merkle tree + STH +
  inclusion proofs deferred to alpha.4.

- **NPS-Node Profile** (`spec/services/NPS-Node-Profile.md`,
  `spec/services/conformance/NPS-Node-L1.md` + L1-CERTIFIED template):
  node-side L1/L2/L3 compliance specification orthogonal to the
  service-side AaaS Profile. NDP `Announce` adds the
  `activation_mode` (`ephemeral` / `resident` / `hybrid`) +
  `activation_endpoint` + `spawn_spec_ref` fields. L1 ships with 21
  `TC-N1-*` test cases + an Ed25519-attested self-attestation
  template.

- **`spec/cr/`** archive directory introduced: pre-1.0 lightweight
  Change Request artifacts complementing the formal RFC track.
  Currently holds `NPS-CR-0001` (implemented) and
  `NPS-CR-0002-anchor-topology-queries` (depends on CR-0001;
  scheduled for alpha.4).

### Version bumps (vs alpha.2)

| Doc | alpha.2 | alpha.3 |
|-----|---------|---------|
| NPS-1 NCP | v0.5 | v0.6 |
| NPS-2 NWP | v0.5 | v0.7 |
| NPS-3 NIP | v0.3 | v0.5 |
| NPS-4 NDP | v0.3 | v0.5 |
| NPS-5 NOP | v0.4 | v0.4 (unchanged) |
| NPS-AaaS-Profile | v0.2 | v0.3 |
| NPS-Node-Profile | n/a | v0.1 (new) |
| frame-registry | v0.5 | v0.9 |
| error-codes | v0.5 | v0.8 |
| status-codes | v0.2 | v0.4 |
| token-budget | v0.2 | v0.2 (unchanged) |

### GitHub Pages

- README + Pages site sources updated to point at the flattened
  `spec/` layout, the new `spec/cr/` and `spec/rfcs/` indexes, and
  v1.0.0-alpha.3 throughout.

---

## [1.0.0-alpha.2] — 2026-04-19

### Changed

- Spec documents resynchronized with the main `LabAcacia/nps` repository:
  - All 11 spec docs advanced from `Draft` → `Proposed`.
  - Version bumps: NPS-0 → v0.3, NPS-1 → v0.5, NPS-2 → v0.5, NPS-3 → v0.3, NPS-4 → v0.3, NPS-5 → v0.4, NPS-Roadmap → v0.3, error-codes → v0.5, status-codes → v0.2, token-budget → v0.2, NPS-AaaS-Profile → v0.2.
  - `frame-registry.yaml` bumped to v0.5, all `protocol_version` entries updated in sync.
- `Depends-On` cross-references updated.

### Deferred

- GitHub Pages content refresh for the alpha.2 release is scheduled for the post-release pass.

---

## [1.0.0-alpha.1] — 2026-04-10

Initial archive of spec documents and GitHub Pages site.

[1.0.0-alpha.4]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.4
[1.0.0-alpha.3]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.3
[1.0.0-alpha.2]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.2
[1.0.0-alpha.1]: https://github.com/LabAcacia/NPS-Release/releases/tag/v1.0.0-alpha.1
