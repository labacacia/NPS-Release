English | [中文版](./NPS-RFC-0003-agent-identity-assurance-levels.cn.md)

---
**RFC Number**: NPS-RFC-0003
**Title**: Three-tier Agent identity assurance levels for anti-scraping / trust gating
**Status**: Accepted (Phase 1 — spec + .NET reference types landed)
**Author(s)**: Ori Lynn <iamzerolin@gmail.com> (LabAcacia)
**Shepherd**: Ori Lynn (pre-1.0 fast-track per `spec/cr/README.md`)
**Created**: 2026-04-21
**Last-Updated**: 2026-04-25
**Accepted**: 2026-04-25 (pre-1.0 fast-track; see `spec/cr/README.md`)
**Activated**: _(set when first reference SDK ships, target v1.0-alpha.3)_
**Supersedes**: _none_
**Superseded-By**: _none_
**Affected Specs**: NPS-3 NIP, NPS-2 NWP, spec/services/NPS-AaaS-Profile.md, spec/error-codes.md, spec/status-codes.md
**Affected SDKs**: .NET, Python, TypeScript, Java, Rust, Go
---

# NPS-RFC-0003: Three-tier Agent identity assurance levels for anti-scraping / trust gating

## 1. Summary

Introduce three **assurance levels** for Agent identities — `anonymous`
(L0), `attested` (L1), `verified` (L2) — and let Nodes declare a
`min_assurance_level` in their NWM. The assurance level travels in
`IdentFrame` and is carried as a critical X.509 extension (see
NPS-RFC-0002 §4.1). This is NPS's answer to **"how do I know a request
is from an Agent, not a scraper?"** — the protocol doesn't block
scrapers, but it gives Nodes a primitive to demand verifiable Agent
provenance and to charge / rate-limit / refuse accordingly.

## 2. Motivation

This RFC responds to a 2026-04-20 review comment:

> 最核心的，如果我要接这套协议，最关心的反而是怎么做反爬？我怎么
> 知道来个请求不是爬虫而是 Agent 访问？

(Most critically — if I adopt this protocol, my biggest concern is
anti-scraping. How do I know a request is an Agent and not a scraper?)

The commenter is right that NPS today answers **"who is this Agent?"**
(NIP) but not **"how much do I trust this Agent's identity?"** An NID
is just a public key — anyone, including a scraper farm, can mint one
and sign requests with it. Without a way to rank identities by
trustworthiness, Nodes either accept anonymous traffic (scraper-friendly)
or hand-maintain allowlists (operational burden).

The X.509 PKI world solved the same class of problem with EV / OV / DV
certificates and the CA/B Forum baseline. The identity-proofing world
solved it with NIST SP 800-63 Identity Assurance Levels (IAL1/2/3). We
adopt the tier shape, not the human-identity specifics.

The concrete business model this unlocks: an AaaS operator can price
`L2`-verified Agent traffic higher than `L0`, auditors can require `L2`
for regulated integrations, and victims of scraper abuse can set
`min_assurance_level: attested` on hot endpoints without breaking
legitimate Agents that upgrade their NIDs.

## 3. Non-Goals

- Does NOT mandate a specific identity-proofing method. CAs decide
  what "attested" or "verified" means in their trust domain, subject
  to the minimum criteria in §4.
- Does NOT replace rate limiting, WAF, CAPTCHA, or other anti-abuse
  layers. Assurance levels are a **complement**, not a substitute.
- Does NOT block scrapers. A Node can still accept `L0` traffic — the
  level is a *signal*, not an enforcement decision by itself.
- Does NOT define reputation. Reputation (has this NID behaved badly
  before?) is NPS-RFC-0004's scope. Assurance level is about the
  *provenance* of the identity claim, not the identity's track record.
- Does NOT handle humans. This is Agent-to-Agent / Agent-to-Node; it
  deliberately avoids FIDO2 / WebAuthn since those require a human at
  the keyboard.

## 4. Detailed Design

### 4.1 Assurance Levels

| Level | Enum | Minimum criteria | Typical use |
|-------|------|-------------------|-------------|
| `anonymous` | 0 | Self-signed NID; no CA involvement; or CA-signed without identity binding | Hobbyist Agents, dev/test, "show me a free quote" read-only endpoints |
| `attested` | 1 | NID signed by an RFC-0002-compliant CA; CA attests possession of the NID private key (ACME `agent-01`); contact email / domain verified by CA | Most production Agents; rate-limit tier default |
| `verified` | 2 | L1 criteria **plus** CA attests the operator's legal identity: corporate registration for org-NIDs, or signed attestation from a known AaaS operator for hosted Agents | Regulated integrations (finance, healthcare), paid premium tiers, high-trust orchestration |

**What "attested" really buys a Node**: the CA has verified that
*someone* controls the NID private key *and* an out-of-band channel
(ACME account email, domain, etc.), so if the Agent misbehaves the CA
can at minimum email a human and at most revoke the cert. L0 offers
neither.

**What "verified" adds on top**: the CA binds the NID to a legal
entity. A Node can then issue an invoice, subpoena the operator, or
make contract-grade claims about who it is interacting with.

### 4.2 Wire Format / Frame Changes

**`IdentFrame` gains one field:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `assurance_level` | `enum { anonymous, attested, verified }` | yes | When the NID's cert carries the `id-nid-assurance-level` extension (NPS-RFC-0002 §4.1), this field MUST carry the same value. **Phase gate**: Phase 1–2 (current) — verifiers SHOULD check and log mismatches, but enforcement is opt-in. Enforcement becomes MUST and triggers `NIP-ASSURANCE-MISMATCH` starting Phase 3 (flag day — see §8.1). |

The cert is the source of truth; `assurance_level` in `IdentFrame` is
redundant convenience for servers that don't want to parse the cert
just to route requests.

**Phase 1–2 (current)**: Verifiers SHOULD check the cert extension and
log any mismatch, but MAY skip enforcement. Implementations that do
enforce MUST close the connection with `NIP-ASSURANCE-MISMATCH`.

**Phase 3 (flag day, see §8.1)**: Verifiers MUST check the cert
extension and MUST close the connection with `NIP-ASSURANCE-MISMATCH`
if the two disagree. This is the hard deadline for completing the
upgrade.

**X.509 extension promotion (Phase 3 — not yet active)**: NPS-RFC-0002
defines `id-nid-assurance-level` (`1.3.6.1.4.1.<PEN>.2.1`) as
**non-critical**. Starting Phase 3 flag day, this RFC promotes it to
**critical** — old verifiers MUST reject certs that carry it as critical
until they upgrade. That's intentional: a Node that enforces
`min_assurance_level` MUST NOT silently accept a verifier that can't
parse the extension. The promotion is NOT yet active in Phase 1–2.

### 4.3 Manifest / NWM Changes

NWM gains one optional top-level field:

```yaml
# /.nwm excerpt
min_assurance_level: attested   # default: anonymous
auth:
  required_scopes: [...]
  min_assurance_level: verified  # per-action override (optional)
```

- Top-level `min_assurance_level` applies to all read paths (`/.schema`,
  `/.actions`, anchor fetches, synchronous `/invoke`).
- Per-action `min_assurance_level` under `auth:` overrides it.
- Default is `anonymous` — this RFC does not change the default trust
  posture; Nodes opt in.
- A Node that receives a request below the required level MUST reject
  with `NWP-AUTH-ASSURANCE-TOO-LOW` and status `NPS-AUTH-FORBIDDEN`,
  and SHOULD include a `hint` pointing to an ACME enrollment URL.

### 4.4 Error Codes

New entries in `spec/error-codes.md`:

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NIP-ASSURANCE-MISMATCH` | `NPS-CLIENT-BAD-FRAME` | `IdentFrame.assurance_level` disagrees with the cert extension |
| `NWP-AUTH-ASSURANCE-TOO-LOW` | `NPS-AUTH-FORBIDDEN` | Request's assurance level < Node's `min_assurance_level` |
| `NIP-ASSURANCE-UNKNOWN` | `NPS-CLIENT-BAD-FRAME` | Cert extension carries a value outside the defined enum |

### 4.5 State Machines / Flows

Rejection path (Node enforces `min_assurance_level: attested`):

```
Agent (L0)                         Node
   │                                 │
   │── HelloFrame ─────────────────→ │
   │── IdentFrame (assurance=0) ───→ │
   │                                 │  check: 0 < 1 (attested)
   │ ←── ErrorFrame ──────────────── │  code: NWP-AUTH-ASSURANCE-TOO-LOW
   │     hint: "https://ca.../acme"  │  status: NPS-AUTH-FORBIDDEN
   │ ←── close                       │
```

Upgrade path is out-of-band: the Agent re-enrolls through an
RFC-0002-compliant CA, receives a new cert with `id-nid-assurance-level`
set to `attested`, and retries the connection.

### 4.6 Backward Compatibility

- Old Agents (no assurance-level field)? They present L0 implicitly.
  Nodes with default `min_assurance_level: anonymous` keep working.
- Old Nodes (no `min_assurance_level` in NWM)? They accept everything
  (default L0). New Agents presenting L2 certs still work.
- The X.509 extension flip from non-critical (RFC-0002) to critical
  (this RFC) is **the** compatibility hazard. Gated by `min_agent_version`
  bump and the 21-day window.

---

## 5. Alternatives Considered

### 5.1 Binary "verified or not" (two tiers)

Drop `attested`; only `unverified` and `verified`.

- **Cost**: collapses the middle ground. Most production Agents would
  sit in `unverified` since legal-entity binding is heavy for hobby /
  dev use. Nodes lose the useful knob.
- **Verdict**: rejected. The L1 tier is where the anti-scraping value
  actually lives — L2 is rare by construction.

### 5.2 Reputation-only (skip levels)

Skip assurance levels; let Nodes rely on NPS-RFC-0004 reputation log
instead.

- **Cost**: reputation only works *after* an incident. Assurance
  levels let a Node say "I only do business with Agents that a CA has
  seen" from day one. The two solve different problems.
- **Verdict**: rejected as substitute; adopted as complement (RFC-0003
  + RFC-0004 together).

### 5.3 Free-form CA policy OIDs

Let CAs define their own policy OIDs à la X.509 certificatePolicies;
Nodes enumerate accepted OIDs in NWM.

- **Cost**: every Node now needs to know every CA's OID catalog.
  Operational nightmare; the reason CA/B Forum baselines exist.
- **Verdict**: rejected. `certificatePolicies` remains usable for
  **finer-grained** CA-specific policy on top of our three tiers, but
  the three tiers are the common vocabulary.

### 5.4 Do nothing

- **Cost**: the 2026-04-20 review comment stands unaddressed. Early
  adopters with scraper exposure (news sites, pricing APIs, SaaS
  directories) can't safely flip on NWP. Business case hole.
- **Verdict**: rejected.

---

## 6. Drawbacks & Risks

- **CA policy burden**: CAs now need a written policy on how they
  assign L1 vs L2. LabAcacia MUST ship a reference policy document
  alongside the .NET NIP CA Server.
- **Trust centralization risk**: `verified` level concentrates power
  in CAs that can attest legal identity. Mitigated by: (a) multiple
  independent CAs supported (no single root); (b) per-Node NWM can
  specify `trusted_issuers` list (RFC-0002 §3 non-goal confirms this
  stays possible).
- **Gaming risk**: a scraper buys an L2 cert from a lax CA. Mitigated
  by: (a) NPS-RFC-0004 reputation log — a misbehaving L2 NID is
  worse for the issuing CA than L0 anonymity; (b) Nodes can maintain
  `trusted_issuers`.
- **Migration cost**: ~1 SDK-week per language (enum + NWM field +
  error handling). Light compared to RFC-0002.
- **Reversibility**: moderate. The enum can be deprecated if the
  model fails; the cert extension flip to critical is the hard part.

---

## 7. Security Considerations

- **Downgrade attack**: attacker presents `IdentFrame.assurance_level=2`
  while the cert extension says 0. Mitigated: `NIP-ASSURANCE-MISMATCH`
  closes the connection; cert is source of truth.
- **Level inflation by issuers**: L2 is only as strong as the weakest
  CA allowed to issue L2. Mitigated by `trusted_issuers` (per-Node)
  and by RFC-0004 reputation tracking per-issuer.
- **No new cryptographic assumption**: reuses RFC-0002's X.509 +
  Ed25519 stack.
- **DoS vector**: none beyond the baseline cert parse cost.
- **Privacy**: L2 certs bind a legal entity. This is a feature for
  regulated use cases, a bug for pseudonymous Agents. The default
  (anonymous) protects pseudonymous Agents; Nodes demanding L2 do
  so deliberately.

---

## 8. Implementation Plan

### 8.1 Phasing

| Phase | Scope | Exit criterion |
|-------|-------|----------------|
| 1 | .NET NIP parses cert extension; IdentFrame carries field; NWM parses `min_assurance_level`; enforcement optional | Unit tests green; the default behavior is unchanged |
| 2 | All 6 SDKs + 6 CA servers; CA servers issue L1 certs via ACME; L2 issuance flow documented per CA | Cross-SDK interop: L0/L1/L2 matrix green |
| 3 | `min_assurance_level` enforcement on by default when set in NWM; extension `id-nid-assurance-level` promoted to critical. **Flag day**: announced via NPS-Dev GitHub Discussions with ≥ 21 calendar days notice before the activation date; after activation, `NIP-ASSURANCE-MISMATCH` enforcement is MUST and non-compliant implementations MUST NOT claim any NPS conformance level | No regressions; all 6 SDKs pass mismatch-enforcement tests |
| 4 | Remove L0-default fast path from Nodes that opted in to stricter defaults | N/A — operators decide |

### 8.2 SDK Coverage Matrix

| SDK | Owner | Status | Notes |
|-----|-------|--------|-------|
| .NET | Ori Lynn | pending | Reference impl |
| Python | _TBD_ | pending | — |
| TypeScript | _TBD_ | pending | — |
| Java | _TBD_ | pending | — |
| Rust | _TBD_ | pending | — |
| Go | _TBD_ | pending | — |

### 8.3 Test Plan

1. L0 Agent against `min_assurance_level: attested` Node → rejected
   with `NWP-AUTH-ASSURANCE-TOO-LOW`.
2. L2 Agent against `min_assurance_level: anonymous` Node → accepted.
3. Assurance mismatch (IdentFrame says L2, cert ext says L0) →
   `NIP-ASSURANCE-MISMATCH`.
4. Per-action override: L1-globally, L2-on-`orders.create` → L1 Agent
   gets `/invoke:orders.create` rejected, other actions pass.
5. Cert without extension against enforcing Node (post-Phase 3) →
   treated as L0 (implicit) and rejected if level > 0.
6. Unknown enum value in cert → `NIP-ASSURANCE-UNKNOWN`.

### 8.4 Benchmarks

- NWM size: +1 optional field, trivial.
- IdentFrame size: +1 byte for the enum + X.509 extension already
  accounted for in RFC-0002's budget.
- Per-request check cost: single integer compare after cert parse.
  Expected overhead < 1 µs.

---

## 9. Empirical Data

None yet. Before `Accepted`, commit:
- A scenario test showing a 10-request loop from L0 and L1 Agents
  against a `min_assurance_level: attested` Node, confirming the L0
  loop gets 403 early and never touches business logic.
- One external CA (likely `pebble`-based test harness) issuing L1
  certs via ACME `agent-01`.

| Metric | Baseline | Proposed | Delta | Method |
|--------|----------|----------|-------|--------|
| NWM size | _no field_ | +~40 B | +~40 B | JSON byte count |
| Per-request overhead | _not enforced_ | ~1 µs | +1 µs | `BenchmarkDotNet` |

---

## 10. Open Questions

- [ ] **OQ-1**: Where does the "reference CA policy document" live?
  Proposed: `tools/nip-ca-server/docs/policy.md`. Owner: Ori Lynn.
- [ ] **OQ-2**: Should L2 certs carry a mandatory legal-entity name
  field (X.509 `O`, `jurisdictionOfIncorporation`)? Default position:
  yes; deferred to CA policy document.
- [ ] **OQ-3**: Does Gateway Node (NPS-AaaS-Profile) promote the
  assurance level of the Agent it fronts, or require the backing
  Action Node to enforce independently? Default: Gateway enforces;
  backing Node trusts the Gateway's decision via `X-NPS-Authed-Nid`
  header. Pending AaaS working-group sign-off.

---

## 11. Future Work

- **NPS-RFC-0004**: NID reputation log (CT-style). Complements
  assurance levels: levels are about *provenance*, reputation is
  about *behavior*.
- Follow-up: bulk-revocation flow when a CA is found to be lax about
  L2 issuance.
- Follow-up: publish a `trusted_issuers` well-known list for AaaS
  profile.

---

## 12. References

- NIST SP 800-63-3 — "Digital Identity Guidelines" (IAL definition)
- CA/B Forum Baseline Requirements — EV / OV / DV tier shape
- RFC 5280 §4.2.1.4 — certificatePolicies
- NPS-RFC-0002 — X.509 + ACME for NID certs (prerequisite)
- NPS-RFC-0004 — NID reputation log (complement, in-flight)
- Discussion: 2026-04-20 review comment on anti-scraping

---

## Appendix A. Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-04-21 | Ori Lynn | Initial draft |
| 2026-04-25 | Ori Lynn | Accepted via pre-1.0 fast-track. Spec changes landed: NPS-3 §5.1.1 Assurance Levels + IdentFrame `assurance_level` field, NPS-2 NWM `min_assurance_level` field, error codes `NIP-ASSURANCE-MISMATCH` / `NIP-ASSURANCE-UNKNOWN` / `NWP-AUTH-ASSURANCE-TOO-LOW`. Phase 1 .NET reference types (`NPS.NIP.AssuranceLevel` enum, `IdentFrame.AssuranceLevel`, `NipVerifyContext.MinAssuranceLevel`, `NeuralWebManifest.MinAssuranceLevel`, related error/status code constants) landed alongside; active enforcement in the verifier remains opt-in per RFC §8.1 (Phase 1 = parse only, default unchanged). Phase 2 (other 5 SDKs + 6 CA Servers issuing L1 certs via ACME — depends on RFC-0002) deferred to v1.0-alpha.4. The X.509 critical-extension flip (§4.2) coordinates with RFC-0002 and is NOT yet active. AaaS-Profile §10 OQ-3 (Gateway enforcement vs backing-Node enforcement) deferred to CR-0001 follow-up. |
