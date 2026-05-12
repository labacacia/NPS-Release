<!--
Copyright 2026 INNO LOTUS PTY LTD
Developed under LabAcacia Open Source Initiative
Licensed under the Apache License, Version 2.0
-->

# NPS Change Request: Wire-in IANA PEN 65715 (replace provisional 1.3.6.1.4.1.99999)

**CR ID**: NPS-CR-0004
**Target version**: v1.0-alpha.6
**Status**: Draft
**Type**: Backward-incompatible OID change (every cert issued under the provisional arc MUST be revoked and re-issued)
**Author**: Ori, LabAcacia
**Affected components**: NIP spec (NPS-3), NPS-RFC-0002, NPS-Roadmap (R08), all 6 SDKs (`dotnet`, `typescript`, `python`, `java`, `go`, `rust`), `tools/nip-ca-server` and its frozen examples (`example/{ts,python,go,rust}`), top-level CHANGELOG / README / CLAUDE.md

---

## 1. Summary

IANA assigned **Private Enterprise Number 65715** to *Neuro Protocol Suites Committee* on **2026-05-08**. This CR wires the assigned PEN into the NPS spec and reference implementations, replacing the provisional / unregistered arc `1.3.6.1.4.1.99999` that NPS-RFC-0002 had been carrying as a documented placeholder under §10 OQ-2.

The change:

- New OID root: `1.3.6.1.4.1.65715`. All NPS-defined sub-OIDs move under this arc with their existing semantics preserved.
- New cert-extension OID `id-nps-node-roles = 1.3.6.1.4.1.65715.2.2` is **reserved** here for the node-roles certificate extension referenced (without an OID) in the NIP v0.6 changelog. No consumer is required to populate it yet — this CR only fixes the namespace.
- NIP spec version bump **0.7 → 0.8** (front matter and version block of `spec/NPS-3-NIP.md` and `spec/NPS-3-NIP.cn.md`).
- NPS-RFC-0002 promotion **Draft → Proposed**, since the only blocker (PEN assignment) is resolved.
- Roadmap R08 (the row tracking RFC-0002 promotion) closes with a pointer to this CR.

This CR documents the change. The actual file edits are sequenced as follow-up tasks under the same alpha.6 milestone — this CR file does NOT mutate code or other spec files.

## 2. Motivation

The provisional arc `1.3.6.1.4.1.99999` was always understood to be temporary and operationally unsafe: it sits inside an unassigned IANA PEN slot, may collide with another organisation's future assignment, and was therefore explicitly forbidden for conformance and production use by NPS-RFC-0002 §10 OQ-2 and the NIP §5.1.1 "attested" assurance-level note.

Now that PEN 65715 is officially assigned:

- The conformance lock-out language in NIP §5.1.1 and the EXPERIMENTAL banner in RFC-0002 are no longer accurate — they describe a hypothetical we are now past.
- Every cert minted by the prototype `nip-ca-server` (and the four frozen example ports) embeds an OID arc that is unsafe to leave in any deployment claiming RFC-0002 compliance.
- Roadmap R08 has been blocked solely on this assignment. Closing it unblocks RFC-0002 promotion and the L1/L2 conformance work that depends on it.

Doing the wire-in as a single coordinated CR (rather than as a drive-by edit) keeps the SDK changes, the spec promotion, and the operational migration note (revoke + re-issue of any prototype certs) auditable and traceable from one document.

## 3. Specification changes

### 3.1 New OID root and sub-tree

The NPS PEN arc moves from `1.3.6.1.4.1.99999` (provisional, unregistered) to **`1.3.6.1.4.1.65715`** (IANA-assigned to *Neuro Protocol Suites Committee*, 2026-05-08).

Sub-tree map (semantics of each existing sub-arc are **preserved verbatim** — this is a namespace move, not a redefinition):

| OID | Symbol | Semantics |
|---|---|---|
| `1.3.6.1.4.1.65715` | NPS PEN root | Root arc owned by Neuro Protocol Suites Committee |
| `1.3.6.1.4.1.65715.1` | `EkuArc` | NPS Extended Key Usages |
| `1.3.6.1.4.1.65715.1.1` | `EkuAgentIdentity` | EKU — cert subject is an NPS Agent (NPS-RFC-0002 §4.1) |
| `1.3.6.1.4.1.65715.1.2` | `EkuNodeIdentity` | EKU — cert subject is an NPS Node (NPS-RFC-0002 §4.1) |
| `1.3.6.1.4.1.65715.1.3` | `EkuCaIntermediateAgent` | EKU — CA may sign `agent-identity` certs (NPS-RFC-0002 §4.1) |
| `1.3.6.1.4.1.65715.2` | `ExtensionArc` | NPS custom certificate extensions |
| `1.3.6.1.4.1.65715.2.1` | `NidAssuranceLevel` | Cert ext — encodes assurance level (0=anonymous, 1=attested, 2=verified) per NPS-RFC-0003 / NIP §5.1.1 |
| `1.3.6.1.4.1.65715.2.2` | `IdNpsNodeRoles` | **(NEW — reserved by this CR)** Cert ext — node-roles encoding referenced by the NIP v0.6 changelog. No OID was previously assigned. Reserved here so SDK constants land alongside `.2.1`; no consumer is mandated by this CR |

Standard X.509 / Ed25519 OIDs (notably `1.3.101.112` from RFC 8410) are unaffected and MUST NOT be touched.

### 3.2 NIP spec edits (deferred to follow-up tasks)

In `spec/NPS-3-NIP.md` and `spec/NPS-3-NIP.cn.md`:

- §5.1.1 "attested" row: remove the conformance lock-out clause that gated implementations behind PEN assignment. Update the cert-extension text to reference `1.3.6.1.4.1.65715.2.1` (assurance-level) and `1.3.6.1.4.1.65715.2.2` (node-roles).
- Front-matter / version block: bump `Version: 0.7` → `Version: 0.8`, set `Date:` to 2026-05-08, in both EN and CN files.
- EN and CN MUST stay in parity (translate any new sentences).

### 3.3 NPS-RFC-0002 status promotion (deferred to follow-up task)

In `spec/rfcs/NPS-RFC-0002-x509-acme-nid-certs.md` and its `.cn.md` sibling:

- Status line: `Draft — EXPERIMENTAL` → `Proposed`.
- Remove or downgrade the EXPERIMENTAL banner blocks that warn against use, since the unregistered-OID hazard is resolved.
- §10 OQ-2: keep the operational requirement that any cert previously issued under `1.3.6.1.4.1.99999.*` MUST be revoked and re-issued under `1.3.6.1.4.1.65715.*` before the issuing deployment can claim RFC-0002 compliance. The PEN-assignment prerequisite text is removed.
- Add a one-line revision-log entry: "Promotion unblocked by IANA PEN 65715 assignment (2026-05-08); see NPS-CR-0004."

### 3.4 Roadmap R08 closure (deferred to follow-up task)

In `spec/NPS-Roadmap.md` and `spec/NPS-Roadmap.cn.md`, the row tracking *RFC-0002 promotion blocked on IANA PEN assignment* MUST be marked closed with a reference to this CR and the assignment date 2026-05-08. The original risk text is preserved for traceability — only its status / resolution column changes.

## 4. SDK and code impact (deferred to follow-up tasks)

This CR enumerates the touch points; the actual edits land in companion tasks under the same alpha.6 milestone.

| Component | Touch points |
|---|---|
| `.NET` SDK (`impl/dotnet/src/NPS.NIP/X509/NpsX509Oids.cs`) | Replace `LabAcaciaPenArc` constant. Reserve `IdNpsNodeRoles = ExtensionArc + ".2"`. Sweep stale comments referring to "provisional", "99999", "MUST be replaced once IANA assigns", etc. (see §5 below) |
| `.NET` builder / verifier source | No runtime behaviour change (constants compose). Comment sweep only. |
| TypeScript SDK (`impl/typescript/src/nip/x509/oids.ts`) | Replace `LAB_ACACIA_PEN_ARC` constant; reserve `ID_NPS_NODE_ROLES_OID` alongside `NID_ASSURANCE_LEVEL_OID`. Update file-header comment. |
| Python SDK (`impl/python/nps_sdk/nip/x509/oids.py`) | Replace `LAB_ACACIA_PEN_ARC` constant; reserve `ID_NPS_NODE_ROLES`. Update module docstring. |
| Java SDK (`impl/java/src/main/java/com/labacacia/nps/nip/x509/NpsX509Oids.java`) | Replace `LAB_ACACIA_PEN_ARC`; reserve `ID_NPS_NODE_ROLES`. Update Javadoc. |
| Go SDK (`impl/go/nip/x509/oids.go`) | Patch each `asn1.ObjectIdentifier{1, 3, 6, 1, 4, 1, 99999, ...}` literal to `... 65715, ...`. Add `OidIdNpsNodeRoles` for `.2.2`. Update file-header comment. |
| Rust SDK (`impl/rust/nps-nip/src/x509/oids.rs`) | Patch each `&[1, 3, 6, 1, 4, 1, 99999, ...]` literal to `... 65715, ...`. Add `ID_NPS_NODE_ROLES_OID` for `.2.2`. Update module docstring. |
| `tools/nip-ca-server/example/ts/src/ca.ts` | Replace literal `"1.3.6.1.4.1.99999.*"` strings with `"1.3.6.1.4.1.65715.*"`. Frozen example — minimum viable change, no refactor. |
| `tools/nip-ca-server/example/python/ca.py` | Same. |
| `tools/nip-ca-server/example/go/ca/ca.go` | Patch each int slice. |
| `tools/nip-ca-server/example/rust/src/ca.rs` | Patch each int slice. |
| `CHANGELOG.md` / `CHANGELOG.cn.md` | Add 2026-05-08 entry summarising the wire-in (PEN, NIP version bump, RFC-0002 promotion, `.2.2` reservation, revoke+re-issue requirement). |
| `README.md`, `CLAUDE.md` | Replace any remaining mention of "provisional PEN", "99999", or "awaiting PEN" with the assigned arc and a one-line pointer to NPS-CR-0004. |

Total expected line delta in code: ~28 occurrences across the 6 SDKs and ~12 occurrences across the 4 frozen example ports (per the scoping pass).

## 5. Comment-hygiene policy for affected files

The X.509 OID source files in every SDK currently carry comments justifying the *provisional* placeholder ("MUST be replaced once IANA assigns the real PEN", "Provisional LabAcacia PEN arc", "1.3.6.1.4.1.99999 arc is provisional pending IANA Private Enterprise Number assignment", etc.). After this CR:

- Where the comment carries useful context beyond the placeholder (e.g. cross-references to RFC-0002 §4.1 or §10 OQ-2), rewrite it to refer to *IANA-assigned PEN 65715* and keep the cross-reference.
- Where the comment exists *only* to justify the placeholder, delete it. The constant value at the assigned arc speaks for itself.
- Do NOT alter runtime behaviour or constant values in the comment sweep — that is the namespace-replacement task.

## 6. Migration impact

**External impact**: any deployment that minted certs under the provisional `1.3.6.1.4.1.99999.*` arc MUST revoke those certs and re-issue under `1.3.6.1.4.1.65715.*` before claiming NPS-RFC-0002 compliance. This mirrors the operational requirement carried in NPS-RFC-0002 §10 OQ-2 and is unconditional: the provisional arc is unregistered, may collide with future IANA assignments, and offers no compliance standing.

There are no production deployments at v1.0-alpha.5 minting under the provisional arc that are not covered by this requirement; the prototype `nip-ca-server` and its four frozen example ports are the only known issuers.

**Internal impact**:

- `nip-ca-server` (`tools/nip-ca-server`) — recompile after Task 2 (.NET SDK constant change); behaviour is otherwise unchanged. Any existing test fixtures storing certs under the provisional arc need regeneration.
- Frozen example ports (`tools/nip-ca-server/example/{ts,python,go,rust}`) — receive a small literal-replacement edit per CLAUDE.md note 8.
- L1 / L2 conformance suites — gain the ability to assert RFC-0002 compliance once RFC-0002 promotes (independent CR is not required for that follow-on; suite text may be updated in the same PR).

**Migration window**: single release. alpha.6 ships the wired-in arc and the RFC-0002 promotion together. There is no transitional dual-OID period — the provisional arc was never valid for conformance or production, so emitting both would be misleading.

## 7. Out of scope (explicit non-changes)

To prevent scope creep, the following are explicitly NOT part of this CR:

- **Rebrand of the `nps-purpose` JWS claim**. The string `"nps-purpose"` continues to be used as documented in NIP §8 and NPS-CR-0003 §3.5. Any naming review for this claim is a separate decision outside this CR.
- **Rebrand of the `urn:nps:` NID namespace**. The `urn:nps:agent:...` / `urn:nps:node:...` URN prefix is unaffected by the PEN assignment and remains the canonical NID namespace per NIP §3. A future URN registration (RFC 8141) would be a separate workstream.
- **Cert extension semantics for `id-nps-node-roles`**. This CR only *reserves* the OID at `.2.2`. The wire format of the extension's ASN.1 value, who emits it, and who consumes it are not specified here — they remain whatever the NIP v0.6 changelog documented and any future CR may further constrain them.
- **Any change to `1.3.101.112` (Ed25519 / RFC 8410)**. That OID is owned by IETF and unaffected.
- **Existing test fixtures under `impl/dotnet/tests/` that hardcode `99999` for unrelated reasons** (e.g. integer clamp tests). Constant replacement is scoped to OID literals only — see Task 2 for the exact grep contract.

## 8. Acceptance criteria

This CR is considered accepted and ready to merge when:

- [ ] This document (`spec/cr/NPS-CR-0004-pen-wirein.md`) is committed under `dev`.
- [ ] All 6 SDKs replace `1.3.6.1.4.1.99999` with `1.3.6.1.4.1.65715` and reserve `.2.2` (`IdNpsNodeRoles`) — Task 2.
- [ ] .NET SDK comment-hygiene sweep complete — Task 3.
- [ ] `spec/NPS-3-NIP.md` and `.cn.md` updated; version 0.7 → 0.8 — Task 4.
- [ ] `spec/rfcs/NPS-RFC-0002-x509-acme-nid-certs.{md,cn.md}` promoted Draft → Proposed — Task 5.
- [ ] Roadmap R08 row closed — Task 6.
- [ ] Four frozen example ports updated — Task 7.
- [ ] CHANGELOG, README, and CLAUDE.md mentions of the provisional PEN replaced — Task 8.
- [ ] `dotnet test` and any CI checks pass against the wired-in arc.
- [ ] At least one independent reviewer (besides the author) signs off.

## 9. CHANGELOG entry (proposed text, EN)

```markdown
## [v1.0-alpha.6] - 2026-05-08

### Spec / RFC

- **PEN wire-in (NPS-CR-0004)**: Replaced provisional / unregistered arc
  `1.3.6.1.4.1.99999` with the IANA-assigned arc `1.3.6.1.4.1.65715`
  (Neuro Protocol Suites Committee, assigned 2026-05-08) across the
  spec, all 6 SDKs, the prototype `nip-ca-server`, and the four frozen
  example ports. Reserved `id-nps-node-roles` at
  `1.3.6.1.4.1.65715.2.2` for the node-roles certificate extension.
  NIP version 0.7 → 0.8. NPS-RFC-0002 promoted Draft → Proposed.
  Roadmap R08 closed. **Operational requirement**: any cert previously
  issued under the provisional arc MUST be revoked and re-issued
  before the issuing deployment may claim RFC-0002 compliance.
```

## 10. References

- IANA PEN registry, assignment 65715 to Neuro Protocol Suites Committee, 2026-05-08
- NPS-RFC-0002 §4.1, §10 OQ-2 — provisional-OID hazard and PEN-assignment prerequisite (this CR resolves the prerequisite)
- NPS-3 NIP §5.1.1 — assurance-level cert-extension reference
- RFC 5280 (X.509), RFC 8410 (Ed25519 in X.509), RFC 8141 (URN)
- `spec/NPS-Roadmap.md` row R08 — closed by this CR
