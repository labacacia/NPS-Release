English | [中文版](./NPS-RFC-0004-nid-reputation-log.cn.md)

---
**RFC Number**: NPS-RFC-0004
**Title**: Append-only NID reputation log (Certificate Transparency for Agents)
**Status**: Accepted (Phase 1 — entry wire format + .NET reference types landed)
**Author(s)**: Ori Lynn <iamzerolin@gmail.com> (LabAcacia)
**Shepherd**: Ori Lynn (pre-1.0 fast-track per `spec/cr/README.md`)
**Created**: 2026-04-21
**Last-Updated**: 2026-05-01
**Accepted**: 2026-04-26 (pre-1.0 fast-track; see `spec/cr/README.md`)
**Activated**: _(set when first reference log operator ships, target v1.0-alpha.4)_
**Supersedes**: _none_
**Superseded-By**: _none_
**Affected Specs**: NPS-3 NIP, NPS-4 NDP, spec/services/NPS-AaaS-Profile.md, spec/error-codes.md
**Affected SDKs**: .NET, Python, TypeScript, Java, Rust, Go
---

# NPS-RFC-0004: Append-only NID reputation log (Certificate Transparency for Agents)

## 1. Summary

Define a Certificate-Transparency-style **append-only log** for NID
behavioral incidents. Entries are signed observations (abuse reports,
rate-limit violations, contract disputes, revocations) that AaaS
gateways, auditors, and CAs publish against specific NIDs. A new
optional NDP sub-path (`/.nid/reputation?nid=...`) lets any party
query the aggregated record. Combined with NPS-RFC-0003 assurance
levels, this gives Nodes both *provenance* (who is this Agent?) and
*track record* (has it misbehaved?).

## 2. Motivation

Follow-up to the same 2026-04-20 review comment that drove RFC-0003.
Assurance levels answer "who is this Agent?" but not "has this
specific NID already caused trouble?" Concretely:

- An `L2` NID that has been revoked for abuse MUST be findable by any
  Node considering whether to transact with it, without requiring
  pre-existing relationship with the revoking CA.
- An AaaS gateway that kicks an Agent off for scraping should be able
  to publish that signal so other gateways can pre-emptively downgrade
  or refuse the NID.
- A Node should be able to distinguish "well-behaved L1 Agent with 2
  years of clean history" from "freshly-minted L1 Agent, 40 rate-limit
  violations in the last 24h."

Existing domain-name reputation systems (Spamhaus, DNSBL) use private
lists. The Certificate Transparency model — publicly auditable,
append-only, tamper-evident, reachable by anyone — maps directly to our
problem because:

- NIDs are already globally unique (pubkey).
- AaaS operators naturally produce these signals; they just need a
  publication standard.
- The log's append-only property limits censorship: a CA can't quietly
  delete a revocation to restore a paying customer's reputation.

## 3. Non-Goals

- Does NOT define what constitutes abuse. Log operators / auditors
  MUST publish their criteria; this RFC defines the wire format only.
- Does NOT require every Node to consult the log. Consultation is
  advisory; enforcement is a per-Node policy decision.
- Does NOT mandate a single canonical log. Multiple independent logs
  are expected; Nodes pick which logs to trust, analogous to CT log
  operators.
- Does NOT include personal data about human operators. Entries refer
  to NIDs and metadata (timestamps, incident type, evidence hashes);
  any doxxing-grade detail belongs out-of-band.
- Does NOT define monetization or ad-hoc blocklists. Commercial
  threat-intel feeds may build on top of the log; that's their layer.

## 4. Detailed Design

### 4.1 Log Entry Format

An entry is a signed JSON object:

```json
{
  "v": 1,
  "log_id": "nid:ed25519:<log-operator-pubkey>",
  "seq": 42817,
  "timestamp": "2026-04-21T14:30:00Z",
  "subject_nid": "nid:ed25519:<agent-pubkey>",
  "incident": "rate-limit-violation",
  "severity": "moderate",
  "window": { "start": "2026-04-21T13:00:00Z", "end": "2026-04-21T14:00:00Z" },
  "observation": {
    "requests": 45000,
    "threshold": 300,
    "endpoint_hash": "sha256:<hash-of-node-origin>"
  },
  "evidence_ref": "https://log.example.com/evidence/42817",
  "evidence_sha256": "hex-of-blob",
  "issuer_nid": "nid:ed25519:<issuer-pubkey>",
  "signature": "base64url(Ed25519(canonical-entry-without-sig))"
}
```

Field semantics:

| Field | Required | Description |
|-------|----------|-------------|
| `v` | yes | Schema version; this RFC defines `1` |
| `log_id` | yes | NID of the log operator appending this entry |
| `seq` | yes | Monotonically-increasing per-log sequence number |
| `timestamp` | yes | RFC 3339 UTC; log operator's clock |
| `subject_nid` | yes | NID this entry is about |
| `incident` | yes | Enum; see §4.2 |
| `severity` | yes | `info` / `minor` / `moderate` / `major` / `critical` |
| `window` | no | Time window the observation covers |
| `observation` | no | Free-form machine-readable detail, incident-type-specific |
| `evidence_ref` | no | URL to richer evidence blob (logs, transcript, etc.) |
| `evidence_sha256` | no | SHA-256 of the evidence blob for tamper detection |
| `issuer_nid` | yes | NID of the party asserting the incident — MAY equal `log_id` |
| `signature` | yes | Ed25519 signature by `issuer_nid`'s private key over canonical form |

**Canonicalization for signing**: JCS (RFC 8785) applied to the entry
object with `signature` omitted. The log operator verifies the issuer
signature before appending and **re-signs** the full entry with their
own key to commit sequence number and timestamp (dual signature:
issuer attests the incident, log operator attests the ordering).

### 4.2 Incident Vocabulary

Initial enum (extensible in follow-up RFCs):

| Value | Meaning |
|-------|---------|
| `cert-revoked` | CA revoked the NID's cert; subject_nid matches revocation |
| `rate-limit-violation` | Sustained violation of published rate limits |
| `tos-violation` | Violated AaaS gateway's published terms |
| `scraping-pattern` | Observed behavior matched scraping heuristics |
| `payment-default` | CGN / fiat payment default on committed transaction |
| `contract-dispute` | Contractual breach on an async NOP task, unresolved |
| `impersonation-claim` | A third party claims subject_nid is impersonating them |
| `positive-attestation` | Explicit positive signal (e.g., audit passed) |

Unknown values MUST be preserved by log operators and returned to
queriers — forward compatibility.

### 4.3 Log Operator Interface

#### 4.3.1 Phase 1 — Submit and Query (current)

A log operator exposes two HTTP endpoints, discoverable via NDP:

```
POST /v1/log/entries        # submit a new entry (requires issuer auth)
GET  /v1/log/entries?nid=<subject_nid>&since=<seq>  # query
```

#### 4.3.2 [Phase 2] — Merkle Integrity Proofs (deferred)

> **[Phase 2 — deferred]** The following endpoints and Merkle structure
> are not part of the Phase 1 implementation; they are targeted for
> v1.0-alpha.5 per §8.1.

```
GET  /v1/log/sth            # signed tree head (Merkle root + seq + timestamp)
GET  /v1/log/proof?seq=<n>&tree_size=<m>  # inclusion proof
```

The Merkle structure mirrors RFC 9162 (CT 2.0): leaves are canonical
entries; internal nodes are SHA-256 hashes; the signed tree head (STH)
commits to the current root and is signed by `log_id`. This gives
queriers cryptographic proof that an entry is included without
downloading the full log.

### 4.4 Manifest / NDP Changes

> **[Phase 2 — deferred]** Everything in this section is targeted for
> v1.0-alpha.5 per §8.1. Neither `/.nid/reputation` nor `reputation_policy`
> is part of the Phase 1 implementation.

NDP gains an optional well-known path for reputation discovery:

```
GET /.nid/reputation?nid=<nid>
    → array of log operator URLs that claim to have entries about this NID
```

This is a *discovery hint*, not a source of truth — Nodes still fetch
entries from the log operators they trust.

NWM optionally declares:

```yaml
# /.nwm excerpt
reputation_policy:
  required_logs: ["log:labacacia-primary", "log:some-industry-consortium"]
  reject_on:
    - { incident: "cert-revoked", severity: ">=minor" }
    - { incident: "scraping-pattern", severity: ">=major", within_days: 30 }
```

Nodes choosing *not* to check reputation simply omit the field.

### 4.5 STH Gossip Protocol

> **[Phase 3 — v1.0-alpha.5]** Enables cross-log consistency verification
> and fork detection.  OQ-1 is resolved in favour of a lightweight NPS-native
> variant (analogous to RFC 9162 §8.1.4 but without the HTTP-header transport
> and with NPS error codes).

#### 4.5.1 Gossip Endpoint

Each log operator MUST expose:

```
GET /v1/log/gossip/sth
```

**Response** (JSON):

```json
{
  "own_sth": {
    "tree_size": 42817,
    "timestamp": "2026-05-01T10:00:00Z",
    "sha256_root_hash": "hex-of-merkle-root",
    "log_id": "nid:ed25519:<log-operator-pubkey>",
    "signature": "base64url(Ed25519(jcs(own_sth without signature)))"
  },
  "peer_sths": [
    {
      "log_id": "nid:ed25519:<peer-pubkey>",
      "received_at": "2026-05-01T09:59:30Z",
      "sth": { /* same shape as own_sth */ }
    }
  ]
}
```

`peer_sths` contains the most recent validated STH received from each
configured peer. Clients querying this endpoint can cross-check peers
without contacting them directly.

#### 4.5.2 Gossip Push Cycle

Log operators configured with a `peers` list MUST run a background
gossip cycle:

1. **Fetch** `GET /v1/log/gossip/sth` from each peer (default interval: 30 s).
2. **Verify** the peer's `own_sth.signature` against the peer's `log_id` public key.
3. **Monotonicity check**: peer's new `tree_size` MUST be ≥ the last accepted `tree_size`
   for that `log_id`.  A decrease is evidence of a fork attempt — log operators
   SHOULD emit a `LOG-FORK-DETECTED` event to local audit and cease transacting
   with that peer until manually reviewed.
4. **Consistency proof** (SHOULD): when the peer's `tree_size` increases, the
   operator SHOULD fetch an RFC 9162 consistency proof from
   `GET /v1/log/proof?from=<prev_size>&to=<new_size>` and verify it before
   accepting the new STH.  Failure to verify = potential fork.
5. **Cache** the accepted peer STH in memory; serve it from `/gossip/sth`.

#### 4.5.3 Configuration

Log operator configuration gains an optional `peers` list:

```json
{
  "peers": [
    { "log_id": "nid:ed25519:<peer>", "endpoint": "https://log2.example.com" }
  ],
  "gossip_interval_s": 30
}
```

`gossip_interval_s` defaults to 30; minimum 10; maximum 3600.

#### 4.5.4 Error Codes (Phase 3 additions)

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NIP-REPUTATION-GOSSIP-FORK` | `NPS-SERVER-INTERNAL` | Cross-peer STH consistency check failed; possible fork detected |
| `NIP-REPUTATION-GOSSIP-SIG-INVALID` | `NPS-CLIENT-BAD-FRAME` | Peer STH signature verification failed |

### 4.7 Error Codes

New entries in `spec/error-codes.md`:

| Error Code | NPS Status | Description |
|------------|------------|-------------|
| `NWP-AUTH-REPUTATION-BLOCKED` | `NPS-AUTH-FORBIDDEN` | Reputation policy matched a reject rule |
| `NIP-REPUTATION-LOG-UNREACHABLE` | `NPS-DOWNSTREAM-UNAVAILABLE` | Required log operator unreachable during policy evaluation |
| `NIP-REPUTATION-ENTRY-INVALID` | `NPS-CLIENT-BAD-FRAME` | Entry signature invalid or canonical form malformed |

### 4.6 State Machines / Flows

Reputation-gated admission:

```
Agent                      Node                         Log Operator
  │                          │                               │
  │── HelloFrame ──────────→ │                               │
  │── IdentFrame (NID) ────→ │                               │
  │                          │ ── GET entries?nid=... ─────→ │
  │                          │ ←── 200 + entries ──────────  │
  │                          │  evaluate reject_on rules     │
  │                          │                               │
  │ ←── (accept | 403) ───── │                               │
```

For hot paths, Nodes SHOULD cache log results with a short TTL
(default 60 s) and refresh asynchronously. A hard `reject_on: cert-revoked`
SHOULD be checked synchronously on every connection but can be
satisfied by OCSP-stapling-like pre-fetch.

### 4.8 Backward Compatibility

- Old Nodes (no `reputation_policy`)? Fully compatible — log is ignored.
- Old Agents? Never affected at the wire level; only their NIDs appear
  in logs.
- NDP sub-path is additive (RFC process requires this per
  `spec/rfcs/README.md`).

---

## 5. Alternatives Considered

### 5.1 Private allowlists per Node

Each Node maintains its own blocklist/allowlist of NIDs.

- **Cost**: operational burden per-Node; duplicated effort; no
  ecosystem-wide signal.
- **Verdict**: rejected as the ecosystem-level solution. Nodes may
  still layer private lists on top.

### 5.2 CA-internal revocation only (no reputation)

Rely solely on NPS-RFC-0002's CRL / OCSP for bad-actor signaling.

- **Cost**: CRL only carries "revoked yes/no", not incident type,
  severity, or observer. Misses the most common signals (rate-limit
  violations, scraping), which aren't revocation-worthy but are
  policy-relevant.
- **Verdict**: rejected as sufficient; CRL remains necessary for
  revocation but is not the full answer.

### 5.3 Centralized single reputation registry

A single LabAcacia-operated registry.

- **Cost**: single point of censorship; LabAcacia becomes a gatekeeper;
  regulatory exposure; conflict of interest (LabAcacia also runs NIP
  CA).
- **Verdict**: rejected. LabAcacia SHOULD operate a reference log but
  MUST NOT be the only one.

### 5.4 Do nothing

- **Cost**: RFC-0003 alone gives Nodes provenance but not behavior
  history. Scraper-at-scale problem partially unsolved.
- **Verdict**: rejected.

---

## 6. Drawbacks & Risks

- **Log-operator abuse**: a rogue operator could flood entries about
  honest NIDs. Mitigation: Nodes only trust logs they opt into; each
  entry carries `issuer_nid` so forged entries are detectable; the
  dual-signature model means the operator can't silently attribute
  entries to third parties.
- **Privacy**: reputation creates permanent history for an NID. An
  Agent wanting a fresh start must rotate NIDs, which also resets
  their assurance level. This is intentional — the cost of a reset
  is the defense against reputation-laundering.
- **Latency**: synchronous log check per connection adds RTT. Mitigated
  by caching, async refresh, and OCSP-style pre-fetch.
- **False positives**: automated scraping-pattern detection is
  imperfect. Mitigation: severity levels + per-Node tunable reject
  rules + `evidence_ref` for dispute.
- **Legal exposure**: operator publishing allegations of misconduct
  invites defamation claims. Mitigation: objective, machine-observable
  incident types preferred; `evidence_ref` strongly recommended.
- **Reversibility**: moderate. Log format is versioned; log operators
  can be deprecated individually without protocol change.

---

## 7. Security Considerations

- **Log tampering**: prevented by Merkle tree + periodic STH
  publication. **[Phase 2]** Nodes SHOULD fetch STH and verify inclusion
  proofs when they download entries.
- **Entry forgery**: impossible without `issuer_nid` private key
  (signature over canonical form). Log operator additionally signs
  for ordering commit.
- **Replay**: `seq` + `timestamp` + log_id uniquely identify an entry;
  inclusion proof binds to a specific tree head.
- **DoS on log operator**: operator MUST rate-limit submissions per
  `issuer_nid` and require authenticated accounts for writes.
- **Censorship**: Merkle tree prevents silent deletion. Operator
  attempting to fork the log is detectable by STH divergence; this
  is the CT "gossip protocol" requirement and SHOULD be implemented
  by queriers cross-checking STHs.
- **Cross-log correlation**: a NID might appear in multiple logs.
  Nodes evaluate each independently per their NWM policy.

---

## 8. Implementation Plan

### 8.1 Phasing

| Phase | Scope | Exit criterion |
|-------|-------|----------------|
| 1 | .NET reference log operator; entry format; submit + query HTTP API; no Merkle proofs yet | Unit tests green; another SDK can query |
| 2 | Merkle tree + STH + inclusion proofs; NDP `/.nid/reputation` in all SDKs; NWM `reputation_policy` parsing | Interop: 2 independent logs + 2 SDK clients cross-check |
| 3 | Default `reputation_policy` for AaaS Profile L2 tier; STH gossip between reference logs (see §4.5) | Logs agree on STH within 60s of commit |
| 4 | Deprecate unsigned-entry experimental flag if any | Clean |

### 8.2 SDK Coverage Matrix

| SDK | Owner | Status | Notes |
|-----|-------|--------|-------|
| .NET | Ori Lynn | ✅ Phase 1+2 done; Phase 3 (gossip) in alpha.5 | Reference log operator also in .NET (`nps-ledger`) |
| Python | _TBD_ | Phase 1+2 pending | Client only |
| TypeScript | _TBD_ | Phase 1+2 pending | — |
| Java | _TBD_ | Phase 1+2 pending | — |
| Rust | _TBD_ | Phase 1+2 pending | — |
| Go | _TBD_ | Phase 1+2 pending | — |

### 8.3 Test Plan

1. Entry round-trip: issuer signs → log operator appends → querier
   validates dual signature.
2. **[Phase 2]** Inclusion proof: query entry `seq=N`, verify against STH at
   `tree_size >= N+1`.
3. **[Phase 2]** Tamper detection: modify entry bytes in storage → STH proof fails.
4. NWM `reject_on` matching: `severity: ">=major"` matches `major`
   and `critical`, not `moderate`.
5. Log unreachable during policy evaluation: `NIP-REPUTATION-LOG-UNREACHABLE`,
   Node falls back per policy (`fail-open` or `fail-closed` configurable).
6. Unknown incident value preserved and returned to querier.

### 8.4 Benchmarks

- Log query latency: target ≤ 20 ms for cached hit, ≤ 200 ms for
  cold fetch including STH verification.
- Entry submission throughput: reference log operator MUST sustain
  1000 entries/s on commodity hardware.
- Memory: Merkle tree for 10M entries ≤ 1 GiB resident.

---

## 9. Empirical Data

None yet. Before `Accepted`:
- Reference .NET log operator implementation.
- End-to-end scenario: AaaS gateway publishes `scraping-pattern` entry;
  second gateway queries and applies `reject_on`.
- Merkle inclusion-proof benchmark.

| Metric | Baseline | Proposed | Delta | Method |
|--------|----------|----------|-------|--------|
| Log query latency (hot cache) | _N/A_ | ≤ 20 ms | — | Wall-clock |
| Entry submit throughput | _N/A_ | ≥ 1000/s | — | Load test |
| Merkle tree size (10M entries) | _N/A_ | ≤ 1 GiB | — | Resident set |

---

## 10. Open Questions

- [x] **OQ-1**: STH gossip protocol — resolved (v1.0-alpha.5): lightweight NPS-native
  variant (analogous to RFC 9162 §8.1.4); see §4.5 for full design.
- [ ] **OQ-2**: Dispute mechanism — can a `subject_nid` post a
  `dispute` entry against an allegation about themselves? Default:
  yes, as `incident: self-dispute` referencing the original `seq`.
- [ ] **OQ-3**: Does the log store evidence blobs or only hashes?
  Default: hashes only; blobs hosted by the issuer at `evidence_ref`.
- [ ] **OQ-4**: Entry TTL / expiration. GDPR-style "right to be
  forgotten" interaction? Default: no expiration; Merkle proofs
  require retention. Regulatory handling deferred.

---

## 11. Future Work

- Follow-up RFC: standardize automated scraping-pattern detection
  criteria so `scraping-pattern` entries are reproducible.
- Follow-up RFC: monetary staking / bonding primitive — an `issuer_nid`
  stakes CGN against an entry; slashed if entry proven false.
- Follow-up RFC: log-operator accreditation tier.

---

## 12. References

- RFC 9162 — "Certificate Transparency Version 2.0"
- RFC 8785 — "JSON Canonicalization Scheme (JCS)"
- RFC 3339 — "Date and Time on the Internet"
- NPS-RFC-0002 — X.509 + ACME for NID certs (cert revocation source)
- NPS-RFC-0003 — Assurance levels (complement: provenance vs. behavior)
- Discussion: 2026-04-20 review comment on anti-scraping

---

## Appendix A. Revision History

| Date | Author | Change |
|------|--------|--------|
| 2026-04-21 | Ori Lynn | Initial draft |
| 2026-04-26 | Ori Lynn | Accepted via pre-1.0 fast-track. Phase 1 spec changes landed: NPS-3 §5.1.2 Reputation Log Entry (12-field signed JSON, 8-value `incident` enum, 5-step `severity` enum, JCS dual-signature rule), error codes `NIP-REPUTATION-ENTRY-INVALID` / `NIP-REPUTATION-LOG-UNREACHABLE` / `NWP-AUTH-REPUTATION-BLOCKED`, new `NPS-DOWNSTREAM-UNAVAILABLE` status code. Phase 1 .NET reference types landed under `NPS.NIP.Reputation.*`. Phase 2 (Merkle tree + STH + inclusion proofs + NDP `/.nid/reputation` discovery + NWM `reputation_policy` parsing) deferred to v1.0-alpha.4 per RFC §8.1. Phase 3 (default policy in AaaS Profile L2 + STH gossip) deferred to alpha.5+. |
| 2026-05-01 | Ori Lynn | Phase 3 spec landed (v1.0-alpha.5): §4.5 STH Gossip Protocol (30s push cycle, `/v1/log/gossip/sth` endpoint, monotonicity + consistency-proof verification, fork detection); OQ-1 resolved; two new error codes `NIP-REPUTATION-GOSSIP-FORK` / `NIP-REPUTATION-GOSSIP-SIG-INVALID`; AaaS-Profile L2 default `reputation_policy` added in `NPS-AaaS-Profile.md`. Phase 3 .NET reference implementation in `nps-ledger`. |
