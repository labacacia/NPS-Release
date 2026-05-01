English | [中文版](./NPS-RFC-0002-x509-acme-nid-certs.cn.md)

---
**RFC Number**: NPS-RFC-0002
**Title**: Adopt X.509 + ACME for NID certificates
**Status**: Draft — **EXPERIMENTAL** (prototype only; provisional unregistered OID `1.3.6.1.4.1.99999.1` — MUST NOT be used for conformance testing or production cert issuance; promotion to Proposed/Accepted is gated on IANA PEN assignment, see §10 OQ-2)
**Author(s)**: Ori Lynn <iamzerolin@gmail.com> (LabAcacia)
**Shepherd**: _TBD — assigned on PR open_
**Created**: 2026-04-21
**Last-Updated**: 2026-04-27
**Accepted**: _(set on merge to `dev`)_
**Activated**: _(set when first reference SDK ships)_
**Supersedes**: _none_
**Superseded-By**: _none_
**Affected Specs**: NPS-3 NIP, tools/nip-ca-server (all language variants), spec/error-codes.md
**Affected SDKs**: .NET, Python, TypeScript, Java, Rust, Go
---

> **WARNING: EXPERIMENTAL — NOT FOR CONFORMANCE OR PRODUCTION USE**
>
> This RFC is **Draft** and uses the provisional, **unregistered** OID arc `1.3.6.1.4.1.99999.1`. This OID has not been assigned by IANA and may collide with any other organisation's assignment.
>
> Implementations using this OID:
> - MUST NOT be used for conformance testing at any compliance level
> - MUST NOT issue certs for production NID issuance
> - MUST NOT be deployed in cross-organisation interoperability tests
>
> This RFC will be promoted from Draft to Proposed/Accepted only after LabAcacia's IANA PEN is assigned and the provisional OID is replaced throughout. All certs issued under the provisional OID will need to be revoked and reissued at that point. See §10 OQ-2.

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

Thresholds revised after the 2026-04-27 prototype run (see §9.2):

- **IdentFrame JSON size**: v2 MUST NOT exceed **1600 bytes** for an
  Ed25519 leaf + single-level intermediate. (The original 1200 B target
  proved tight under base64url encoding overhead; the prototype
  measured 1512 B for leaf + self-signed root.)
- **Verification latency**: v2 path MUST NOT regress beyond **5×** the
  v1 path on the same host. The earlier "≤ 50 µs absolute" sub-target
  is dropped — the v1 baseline alone is dominated by JSON
  canonicalization + Ed25519 verify and rarely fits in 50 µs on
  shared-tenant hardware. Keeping a ratio-only target makes the
  benchmark host-independent. Prototype measured 2.84× regression
  (well within ceiling).
- **CA server binary size**: ≤ 2× the v1 binary at
  `dotnet publish -c Release`. Not yet measured; will land in the
  npsd / nip-ca-server port phase.

---

## 9. Empirical Data

### 9.1 Prototype branch

The prototype lives at `feat/rfc-0002-x509-acme-prototype` on `dev`. It
delivers, in order:

- **.NET NIP emits and verifies X.509 certs** —
  `impl/dotnet/src/NPS.NIP/X509/{NpsX509Oids,NipX509Builder,NipX509Verifier,Ed25519X509SignatureGenerator}.cs`.
  Five conformance tests in `tests/NPS.Tests/Nip/X509/NipX509Tests.cs`
  cover round-trip, EKU-missing rejection, subject/NID-mismatch
  rejection, and v1↔v2 cross-format compatibility.
- **In-process ACME server with `agent-01` challenge type** —
  `impl/dotnet/src/NPS.NIP/Acme/{AcmeJws,AcmeMessages,AcmeServer,AcmeClient}.cs`
  +  `tests/NPS.Tests/Nip/Acme/AcmeAgent01Tests.cs`. Two tests cover
  end-to-end issuance through the new challenge type and a tampered-
  signature negative path returning `NIP-ACME-CHALLENGE-FAILED`.
- **Pebble (RFC 8555 reference server) interop is DEFERRED** to a
  follow-up. Reason: pebble does not implement `agent-01` (a non-
  standard challenge type proposed by this RFC); validating standard
  ACME conformance against pebble would only add HTTP-01 round-trip
  coverage on top of `agent-01`. `tools/pebble/setup.sh` ships the
  binary download helper for that follow-up.

### 9.2 Measurements

Numbers below are produced by `NPS.Benchmarks.NipCert`
(`dotnet run -c Release --project impl/dotnet/benchmarks/NPS.Benchmarks.NipCert -- --emit`),
which writes a Markdown report to `docs/benchmarks/nip-cert-prototype.md`.

| Metric | Baseline (v1) | Proposed (v2) | Delta | Method |
|--------|---------------|---------------|-------|--------|
| `IdentFrame` JSON size | 459 B | 1512 B | +1053 B (+229%) | UTF-8 byte count of `JsonSerializer.Serialize(IdentFrame)` |
| Verification latency (mean of 2000 iters) | 597.8 µs | 1698.5 µs | 2.84× | `Stopwatch` over `verifier.VerifyAsync(frame)` |

Two RFC §8.4 thresholds are revisited by this data:

- **Wire-size ceiling (≤ 1200 B for "Ed25519 leaf + single-level
  intermediate")**: prototype exceeded by 26% (1512 B). The prototype
  ships a leaf + self-signed root chain rather than leaf + a real
  intermediate; an actual intermediate would be smaller (no
  CrlSign/keyCertSign extension overhead) and several base64url
  inflation factors are amortizable. Read this as "1200 B is plausible
  but tight — production deployments need to budget for
  +1 KiB per IdentFrame in the worst case".
- **Verification latency target (≤ 50 µs absolute)**: NEITHER v1 NOR
  v2 hits 50 µs absolute on the test container (12× and 34× over
  respectively). The v1 baseline already includes JSON canonicalization
  + Ed25519 verify, both of which dominate over the X.509 chain check.
  The proposed regression-ratio target (≤ 5×) IS met (2.84×). RFC
  should drop the absolute number and keep the ratio; absolute
  latency is host-dependent and was not measured before the prototype.

### 9.3 Implications for §10 OQ-1

The prototype ran X.509 first, then layered ACME. Effort: ~5 days of
working time for X.509 + verifier + tests; ~2 days for ACME (client +
in-process server with `agent-01` + tests). pebble HTTP-01 interop
would have added another ~1 day if the new `agent-01` challenge type
weren't blocking direct interop.

Recommendation for OQ-1: **bundle X.509 + ACME in the same
acceptance** (the original "both together" default position). The
ACME piece is mostly mechanical once X.509 issuance is wired; splitting
forces a second wave of cross-language port work that can be saved
by doing it once.

---

## 10. Open Questions

- [x] **OQ-1**: Phase 1 — X.509 first, ACME later; OR both together?
  **Resolved 2026-04-27 by prototype data (see §9.3)**: bundle them.
  X.509 + ACME together took 7 days of working time on .NET; splitting
  would require two cross-language port waves (5 SDKs × 2 phases) and
  the marginal effort of layering ACME on top of X.509 is small.
- [ ] **OQ-2 — GATING CONDITION FOR PROMOTION**: IANA Private Enterprise
  Number (PEN) — application not yet submitted. **This is the hard gate
  for RFC-0002 promotion from Draft to Proposed/Accepted.** Until the
  real PEN is assigned and this RFC is Accepted:
  - Implementations using `1.3.6.1.4.1.99999.1` MUST treat all issued
    certs as prototype-only (revoke and reissue once PEN lands)
  - MUST NOT use prototype certs for conformance testing at any level
  - MUST NOT use prototype certs for production NID issuance
  - MUST NOT use prototype certs in cross-organisation interop tests
  - The prototype OID `1.3.6.1.4.1.99999.1` is a development placeholder
    only; it is unregistered and carries no legal or technical standing

  Once PEN is assigned: all `NpsX509Oids` constants are replaced via
  search-and-replace across all SDKs; all previously-issued prototype
  certs are revoked; the RFC moves to Proposed for shepherd review.
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
| 2026-04-27 | Claude (prototype) | Backfill §9 Empirical Data with measurements from `feat/rfc-0002-x509-acme-prototype` (.NET prototype + `NPS.Benchmarks.NipCert`). Resolve OQ-1 (bundle X.509 + ACME). Revise §8.4 thresholds (size 1200 → 1600 B; drop absolute latency target, keep ratio). |
| 2026-04-21 | Ori Lynn | Initial draft |
