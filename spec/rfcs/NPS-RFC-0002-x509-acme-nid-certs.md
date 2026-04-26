English | [中文版](./NPS-RFC-0002-x509-acme-nid-certs.cn.md)

---
**RFC Number**: NPS-RFC-0002
**Title**: Adopt X.509 + ACME for NID certificates
**Status**: Draft
**Author(s)**: Ori Lynn <iamzerolin@gmail.com> (LabAcacia)
**Shepherd**: _TBD — assigned on PR open_
**Created**: 2026-04-21
**Last-Updated**: 2026-04-21
**Accepted**: _(set on merge to `dev`)_
**Activated**: _(set when first reference SDK ships)_
**Supersedes**: _none_
**Superseded-By**: _none_
**Affected Specs**: NPS-3 NIP, tools/nip-ca-server (all language variants), spec/error-codes.md
**Affected SDKs**: .NET, Python, TypeScript, Java, Rust, Go
---

# NPS-RFC-0002: Adopt X.509 + ACME for NID certificates

## 1. Summary

Replace the NIP custom certificate format with **X.509v3** certificates
carrying a LabAcacia-registered Extended Key Usage (EKU) OID for
"agent-identity", and move NIP CA issuance from the bespoke REST API to
**ACME (RFC 8555)** with an `agent-01` challenge type suited to
programmatic NID control. Ed25519 remains the primary signature
algorithm (RFC 8410 — Ed25519 in X.509). CRL / OCSP revocation semantics
are preserved.

## 2. Motivation

This RFC responds to a 2026-04-20 review comment: **"CA 不如直接兼容
现有的 CA 套路"** (CAs should be compatible with existing CA practice).

The commenter's point stands. Current NIP CA (`tools/nip-ca-server-*`)
replicates the operational model of a classic X.509 CA — CSR → issuance
→ CRL/OCSP revocation → renewal — but uses a bespoke serialization.
This creates two unnecessary costs:

1. **No tooling reuse.** OpenSSL, rustls, BouncyCastle, `step-ca`,
   HashiCorp Vault PKI, HSM vendors, cert-manager for Kubernetes —
   none of them can sign, validate, or store NIP certs without custom
   integration. The NIP CA server variants in 6 languages each
   re-implement the ceremony.
2. **No issuance-protocol reuse.** Manual NID issuance via a REST API
   doesn't benefit from the operational maturity that ACME has built:
   automated renewal, staging environments, rate-limited public CAs,
   Certbot/lego clients, Kubernetes operators.

Switching to X.509 + ACME costs us ASN.1 parser exposure and ~2–4× cert
size, but buys us 15 years of PKI tooling and the ability to run
"Let's Encrypt for Agents" as a future public service — or to delegate
NID issuance to existing CAs via cross-signing.

## 3. Non-Goals

- Does NOT change the wire format of `IdentFrame` — the certificate
  still travels as `cert_chain` bytes; only the internal encoding
  shifts from NIP-proprietary to X.509.
- Does NOT mandate public CA trust roots. LabAcacia's own CA remains
  the default root; nothing precludes organizations from running
  private X.509 roots.
- Does NOT change `NID` format (`nid:{algo}:{base64url(pubkey)}`).
- Does NOT deprecate Ed25519. Ed25519 stays the primary; RSA / ECDSA-P256
  remain supported where X.509 allows.

## 4. Detailed Design

### 4.1 Wire Format / Frame Changes

**`IdentFrame.cert_chain` encoding** changes from `nip-cert-v1` (custom)
to **DER-encoded X.509 certificate chain** concatenated or length-prefixed
per rule below.

**Subject field mapping:**

| X.509 Field | NIP Value |
|-------------|-----------|
| Subject `CN` | NID string (`nid:ed25519:...`) |
| Subject Alternative Name (SAN) URI | Same NID as a `URI` entry for RFC 5280 compliance |
| Issuer | Issuing CA's NID |
| NotBefore / NotAfter | Standard X.509 validity period |
| Public Key | Ed25519 (RFC 8410 OID `1.3.101.112`) |
| Serial Number | 128-bit random, per CA/B Forum baseline |

**Critical extension — Extended Key Usage:**

Register LabAcacia IANA Private Enterprise Number (PEN) — once assigned,
reserve OID arc `1.3.6.1.4.1.<PEN>.1`.

| OID | Meaning |
|-----|---------|
| `1.3.6.1.4.1.<PEN>.1.1` | `agent-identity` — this cert's subject is an NPS Agent |
| `1.3.6.1.4.1.<PEN>.1.2` | `node-identity` — this cert's subject is an NPS Node |
| `1.3.6.1.4.1.<PEN>.1.3` | `ca-intermediate-agent` — this CA may sign `agent-identity` certs |

EKU is **marked critical**. A verifier that doesn't recognize these
EKUs MUST reject the certificate for NIP purposes. (Generic TLS clients
would also reject, which is intentional — NID certs MUST NOT be mistaken
for TLS server certs.)

**Custom non-critical extension — `nid:assurance-level`:**

Reserved for NPS-RFC-0003. This RFC defines the encoding shape only:

```
nid-assurance-level EXTENSION ::= {
    SYNTAX NidAssuranceLevel
    IDENTIFIED BY id-nid-assurance-level  -- 1.3.6.1.4.1.<PEN>.2.1
}
NidAssuranceLevel ::= ENUMERATED {
    anonymous (0),
    attested  (1),
    verified  (2)
}
```

Non-critical so v0.1 verifiers ignore the field; RFC-0003 flips it to
critical when assurance-level enforcement lands.

### 4.2 Manifest / NWM Changes

None. X.509 cert chain travels inside `IdentFrame`, transparent to NWM.

### 4.3 Error Codes

New entries in `spec/error-codes.md`:

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NIP-CERT-FORMAT-INVALID` | `NPS-CLIENT-BAD-FRAME` | Cert chain is not DER-encoded X.509 or fails ASN.1 parsing |
| `NIP-CERT-EKU-MISSING` | `NPS-CLIENT-BAD-FRAME` | Required NPS EKU (`agent-identity` or `node-identity`) absent |
| `NIP-CERT-SUBJECT-NID-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | Cert Subject CN / SAN URI does not match the NID in IdentFrame |
| `NIP-ACME-CHALLENGE-FAILED` | `NPS-CLIENT-BAD-FRAME` | ACME `agent-01` challenge validation failed |

### 4.4 State Machines / Flows

**ACME `agent-01` challenge** — a new challenge type for NID identity
validation (parallels HTTP/DNS/TLS-ALPN challenges in RFC 8555 §8):

```
Client (Agent)                 ACME Server (NIP CA)
    │                                 │
    │── newAccount (Ed25519 JWK) ──→ │
    │ ←── 201 + kid ────────────────  │
    │                                 │
    │── newOrder (identifiers) ────→ │
    │          identifier: type=nid,  │
    │          value=nid:ed25519:... │
    │ ←── order + authz URL ──────── │
    │                                 │
    │── GET authz URL ──────────────→ │
    │ ←── challenge: type=agent-01,   │
    │     token=T ──────────────────  │
    │                                 │
    │   (Agent signs `T` with its     │
    │    NID private key; serves      │
    │    the signature at a           │
    │    well-known /nip/auth/T       │
    │    on its announced endpoint    │
    │    — OR — posts back to ACME    │
    │    as a JWS)                    │
    │                                 │
    │── POST challenge (signed T) ─→ │
    │ ←── 200 + status=valid ──────── │
    │                                 │
    │── finalize (CSR) ─────────────→ │
    │ ←── 200 + cert URL ──────────── │
    │── GET cert URL ───────────────→ │
    │ ←── 200 + X.509 DER ──────────  │
```

The `agent-01` challenge proves possession of the NID private key in a
way that mirrors TLS-ALPN-01's simplicity: one signed token, no external
DNS / HTTP dependency. Servers MUST implement timing-safe comparison per
`spec/NPS-3-NIP.md §10.2`.

### 4.5 Backward Compatibility

- Old Agents verifying new certs? **Break.** X.509 parser is a prerequisite.
  6 SDKs each gain an X.509 dependency (already present as transitive
  dep for TLS in every language).
- Old CAs issuing old-format certs? **Break.** The CA tool must migrate
  across all 6 language variants.
- Wire-level version detection: IdentFrame gains a `cert_format` field
  (`v1-proprietary` | `v2-x509`). Deployments running v0.2 stacks can
  phase the rollout.
- `min_agent_version` bump with 21-day window per RFC process.

---

## 5. Alternatives Considered

### 5.1 Keep proprietary format, add X.509 export shim

Keep current cert format; offer a one-way export to X.509 for interop
(e.g., `nipc export --x509`).

- **Cost**: two parallel formats forever. Double maintenance for 6 CA
  servers + 6 SDKs.
- **Benefit**: no wire break.
- **Verdict**: rejected. We chose a bespoke format for early velocity;
  X.509 is the right target now that the model is validated.

### 5.2 Adopt X.509 without ACME (keep REST issuance)

Use X.509 but keep the current bespoke `/certs/issue` REST endpoint.

- **Cost**: loses ACME's renewal automation, rate-limiting discipline,
  and rich client ecosystem (Certbot, lego, acme.sh, cert-manager).
- **Benefit**: simpler initial migration (1 spec change instead of 2).
- **Verdict**: rejected as the final state; may be adopted as an
  intermediate step in Phase 1 if engineering bandwidth is tight.
  Flagged as OQ-1.

### 5.3 Do nothing

- **Cost**: 6-language CA servers keep diverging from industry practice.
  Every new deployment is custom integration. Public-CA option forever
  closed off.
- **Verdict**: rejected.

---

## 6. Drawbacks & Risks

- **Migration cost**: ~2 SDK-weeks per language (X.509 parser +
  generator + EKU validation). ACME client: ~1 SDK-week per language.
  CA side: reuse existing ACME server frameworks (`step-ca`, `boulder`,
  `pebble`) for the .NET variant at minimum, then port.
- **Attack surface**: ASN.1 parsers are historically CVE-prone. Mitigation:
  mandate a vetted library (`System.Security.Cryptography.X509Certificates`
  for .NET, `cryptography` for Python, `x509-parser` for Rust, etc.) —
  never hand-roll.
- **Cert size**: NIP proprietary cert ~200 bytes; X.509 minimum ~450 bytes,
  typical ~800 bytes. +600 bytes per identity claim. Trivial in HTTP
  mode; non-trivial for native mode at high frame rates.
- **Ecosystem fragmentation**: brief period during Phase 1–2 where v1
  and v2 cert formats coexist. Mitigated by `cert_format` version field.
- **Reversibility**: hard. Rolling back means removing X.509 support
  across 12 repos. Cost: ~6 SDK-weeks.

---

## 7. Security Considerations

- **ASN.1 parser exposure**: new attack class. Mitigated by requiring
  platform-native parsers (not hand-roll), not honoring unknown
  critical extensions, and rejecting certs with known pathological
  constructs (BMPString, negative integers in fields that don't allow
  them, etc.).
- **EKU-critical discipline**: NID certs MUST mark EKU critical. This
  prevents the classic "cert intended as agent-identity accepted as
  TLS server cert" cross-purpose attack. Verifiers MUST check EKU
  before any other use.
- **ACME account rotation**: RFC 8555 `newAccount` gives account holders
  rotation keys. NIP CA MUST enforce that Ed25519 is the account-key
  algorithm (matches NID algorithm family).
- **Certificate Transparency**: not in this RFC. RFC-0004 adds a
  transparency log. In the interim, NIP CA MUST maintain an
  append-only issuance log queryable by NID.
- **Downgrade attack**: old Agent presenting v1 cert to a v2-capable
  server MUST be rejected once Phase 3 lands. Until then servers accept
  both with version-tagged telemetry.

---

## 8. Implementation Plan

### 8.1 Phasing

| Phase | Scope | Exit criterion |
|-------|-------|----------------|
| 1 | .NET NIP + .NET CA server emit + accept X.509; ACME `agent-01` in .NET CA; v1 + v2 coexist | Unit tests green; cross-format interop (v1 client ↔ v2 server and vice versa) |
| 2 | All 6 SDKs + 6 CA servers support X.509 + ACME; `cert_format=v2` default-off | Cross-SDK cert-accept matrix green; ACME issuance works across all 6 CA variants |
| 3 | Flip `cert_format=v2` default-on; 21-day deprecation notice for v1 | No regressions for 1 release cycle |
| 4 | Remove v1 codepath | v1 support removed from all 12 repos |

### 8.2 SDK Coverage Matrix

| SDK | Owner | Status | Notes |
|-----|-------|--------|-------|
| .NET | Ori Lynn | pending | Primary reference |
| Python | _TBD_ | pending | Use `cryptography` lib |
| TypeScript | _TBD_ | pending | Use `@peculiar/x509` |
| Java | _TBD_ | pending | Bouncy Castle |
| Rust | _TBD_ | pending | `x509-parser` + `rcgen` |
| Go | _TBD_ | pending | stdlib `crypto/x509` |

### 8.3 Test Plan

1. X.509 round-trip: CA issues v2 cert with `agent-identity` EKU; Agent
   presents it; verifier accepts.
2. EKU-critical enforcement: v2 cert without EKU → `NIP-CERT-EKU-MISSING`.
3. Subject / NID mismatch: cert Subject CN != IdentFrame.nid →
   `NIP-CERT-SUBJECT-NID-MISMATCH`.
4. ACME `agent-01` happy path end-to-end.
5. ACME replay protection: repeated challenge token → 400.
6. Cross-format: v1 client + v2 server; v2 client + v1 server (both
   during Phase 2; both fail cleanly during Phase 3+).

### 8.4 Benchmarks

- IdentFrame size: expected +600 bytes. Add regression threshold to
  wire-size benchmark: "IdentFrame-v2 MUST NOT exceed 1200 bytes for
  Ed25519 leaf + single-level intermediate".
- Verification latency: new benchmark measuring X.509 validation path.
  Target: ≤ 50 µs per cert on typical hardware (current proprietary
  parser is ~10 µs; ≤ 5× regression is acceptable given ecosystem win).

---

## 9. Empirical Data

None yet. Before `Accepted`, will commit:
- A prototype branch showing .NET NIP emitting X.509 certs
- Wire-size measurement for v1 vs v2 certs
- ACME `agent-01` interop proof with `pebble` (ACME test server)

| Metric | Baseline (v1) | Proposed (v2) | Delta | Method |
|--------|---------------|---------------|-------|--------|
| Cert size (Ed25519 leaf) | ~200 B | ~500 B | +300 B | DER byte count |
| Verification latency | TBD | TBD | target ≤ 5× | `BenchmarkDotNet` |
| CA server binary size | TBD | TBD | target ≤ 2× | `dotnet publish -c Release` |

---

## 10. Open Questions

- [ ] **OQ-1**: Phase 1 — X.509 first, ACME later; OR both together?
  Owner: Ori Lynn. Target: resolved before Accepted. _Default position:
  both together; ACME implementation effort is small compared to the
  ceremony mindset shift._
- [ ] **OQ-2**: IANA Private Enterprise Number (PEN) — application
  pending. Target: submit within 2 weeks of this RFC entering Accepted.
  Until PEN assigned, use OID arc under `1.3.6.1.4.1.99999.1` (marked
  "PROVISIONAL").
- [ ] **OQ-3**: Should NIP CA offer cross-signing with public CAs
  (e.g., Let's Encrypt)? Defer to a follow-up RFC.

---

## 11. Future Work

- **Follow-up RFC**: Certificate Transparency for NID issuance (overlaps
  with NPS-RFC-0004 scope).
- **Follow-up RFC**: Cross-signing policy with public CAs.
- **Follow-up RFC**: HSM/TPM binding for NID private keys.

---

## 12. References

- RFC 5280 — "Internet X.509 Public Key Infrastructure Certificate and CRL Profile"
- RFC 8410 — "Algorithm Identifiers for Ed25519, Ed448, X25519, and X448"
- RFC 8555 — "Automatic Certificate Management Environment (ACME)"
- RFC 6960 — "X.509 OCSP"
- `spec/NPS-3-NIP.md` — current NIP spec
- `tools/nip-ca-server/README.md` — current CA server
- Discussion: 2026-04-20 review comment on CA compatibility

---

## Appendix A. Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-04-21 | Ori Lynn | Initial draft |
